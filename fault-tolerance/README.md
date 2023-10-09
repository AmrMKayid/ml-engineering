# Fault Tolerance

Regardless of whether you own the hardware or renting it by the hour, in this every speeding up domain of ML, finishing the training in a timely matter is crucial for success. As such if while you were asleep one of the GPUs failed or the disc run out of space and the training crashed, you'd have lost many training hours upon waking.

While during the prohibitively high cost of ML hardware, it'd be very difficult to provide redundancy fail-over solutions as it's done in web-services, nevertheless making your training fault-tolerant is achievable with just a few simple recipes.

As most serious training jobs are performed in a SLURM environment, it'll be mentioned a lot, but a lot of this chapter's insights are applicable to any other training environments.

## Always plan to have more nodes than needed

The reality of the GPU devices is that they tend to fail. Sometimes they just overheat and shut down, but can recover, at other times they just fail and require a replacement.

The situation tend to ameliorate as you use the same nodes for months as the bad apples get gradually replaced, but if you lucky to get a new shipment of GPUs and especially the early GPUs when the technology has just come out, expect a sizeable proportion of those to fail.

Therefore, if you need 64 nodes to do your training, make sure that you have a few spare nodes and study how quickly you can replace failing nodes should that be not enough.

It's hard to predict what the exact redundancy percentage should be, but 5-10% shouldn't be unreasonable. The more you're in a crunch to complete the training on time, the higher the safety margin should be.

Once you have the spare nodes available, validate that your SLURM environment will automatically remove any problematic nodes from the pool of available nodes so that it can automatically replace the bad nodes with the new ones.

If you use a non-SLURM sheduler validate that it too can do unmanned bad node replacements.

You also need at least one additional node for running various preventative watchdogs, possibly offloading the checkpoints and cleanup jobs, all of which are discussed later in this chapter.



## Queue up multiple training jobs

The next crucial step is to ensure that if the training crashed, there is a new job lined up to take place of the previous one.

Therefore, when a training is started, instead of using
```
sbatch train.slurm
```

You'd want to replace that with:

```
sbatch --array=1-10%1 train.slurm
```

This tells SLURM to book a job array of 10 jobs, and if one of the job completes normally or it crashes, it'll immediately schedule the next one.

footnote: `%1` in `--array=1-10%1` tells SLURM to launch the job array serially - one job at a time.

If you have already started a training without this provision, it's easy to fix without aborting the current job by using the `--dependency` flag:

```
sbatch --array=1-10%1 --dependency=CURRENTLY_RUNNING_JOB_ID train.slurm
```
So if your launched job looked like this:

```
$ squeue -u `whoami` -o "%.10i %9P %20j %.8T %.10M %.8l %.6D %.20S %R"
  JOBID    PARTITION NAME             STATE       TIME   TIME_LIM    NODES  START_TIME NODELIST(REASON)
    100    prod      my-training-10b  RUNNING 2-15:52:19 1-16:00:00   64    2023-10-07T01:26:28 node-[1-63]
```
You will now do:
```
sbatch --array=1-10%1 --dependency=100 train.slurm
```
and then the new status will appear as:
```
$ squeue -u `whoami` -o "%.10i %9P %20j %.8T %.10M %.8l %.6D %.20S %R"
     JOBID PARTITION NAME             STATE       TIME   TIME_LIM    NODES  START_TIME NODELIST(REASON)
      100    prod    my-training-10b  RUNNING 2-15:52:19 1-16:00:00   64    2023-10-07T01:26:28 node-[1-63]
101_[10%1]   prod    my-training-10b  PENDING       0:00 1-16:00:00   64                    N/A (Dependency)
```
So you can see that an array of 10 jobs was appended to be started immediately after the current job completes or fails.

Granted that if the condition that lead to the crash is still there the subsequent job will fail as well. For example, if the storage device is full, no amount of restarts will allow the training to proceed. And we will discuss later how to avoid this situation.

But since the main reason for training crashes is failing gpus, ensuring that faulty nodes are automatically removed and the new job starts with a new set of nodes makes for a smooth recovery from the crash.

In the SLURM lingo, the removed nodes are given a new status called `drained`. Here is an example of a hypothetical SLURM cluster:

```
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
prod*       up   infinite       4  drain node-[0-3]
prod*       up   infinite      47  alloc node-[4-51]
prod*       up   infinite      23   idle node-[52-73]
```

Here we have 47 nodes being used (`alloc`, 23 available (`idle`) and 4 unavailable (`drained`).

The sysadmin periodically checks the drained nodes, fixes or replaces them and then makes them again available to be used by changing their state to `idle`.



## Frequent checkpoint saving

Whenever the training job fails, many hours of training can be lost. This problem is mitigated by a frequent checkpoint saving. When the training is resumed it'll continue from the last checkpoint saved. If the failure occurred 12 hours after the last checkpoint has been saved, 12 hours of training is lost and needs to be re-done.

In theory one could save a checkpoint every 10 minutes and only ever lose 10 minutes of training time, but this too would dramatically delay the reaching of the finish line because large models can't be saved quickly and if the saving time starts to create a bottleneck for the training this approach becomes counterproductive.

Depending on your checkpointing methodology and the speed of your IO storage device the saving of a large model can take from dozens of seconds to several minutes. Therefore, the optimal approach lies somewhere in the middle.

The math is quite simple - measure the amount of time it takes to save the checkpoint, multiply it by how many times you'd want to save it and see how much of an additional delay the checkpoint saving will contribute to the total training time.

Use case: While training BLOOM-176B we had an incredibly fast GPFS over NVME filesystem and it took only 40 seconds to save a 329GB checkpoint written concurrently on 384 processes. We saved a checkpoint approximately every 3 hours. As we trained for about 3 months, that means that we saved about 720 times (`90 days * 24h / 3h`) - that is an additional 8 hours was spent just saving the checkpoints (`720 times * 40 secs / 3600 secs`) - or ~0.37% of the training time (`8h / (90 days * 24 hours)`. Now say if the IO were to be 5 times slower, which is very uncommon on the cloud unless one pays for premium IO, that would have become 2% of the training time, which would be quite significant.

footnote: If you don't have a large local storage and you have to offload the checkpoints to the cloud, make sure that the 2 most frequent checkpoints remain local to allow for quick resume. The reason for two is that it's possible that the very last checkpoint got corrupted or didn't finish saving if a crash occurred during saving.


## Kill switch

In many SLURM environments users have no `sudo` access and when one user started a training and went to sleep, and then a problem has been discovered, the other users can't easily stop the training and restart it again.

This was the situation during BLOOM-176B training and we implemented a kill-switch to handle that. The mechanism is very simple. The training loop polls for a specific file to appear before starting a new iteration and if the file is there the program saves the checkpoint and exits, allowing users other than the one who started the previous training to change things and restart it again. And additional poll was added at the very beginning of `main` so that if there was a long job array queued by the user who is asleep they could be "burned through" quickly by getting each job exit quickly on start.

This is also discussed [here](../slurm#overcoming-the-lack-of-group-slurm-job-ownership).

## Save switch

Similar to the kill switch a save switch is a variation of the former where instead of stopping the training, if the training loop discovers that a save-switch file appears - it will save a checkpoint, but will continue training. It'll also automatically remove the save-switch from the file-system, so that it won't accidentally start saving a checkpoint after every iteration.

This feature can be very useful for those who watch the training charts. If one sees an interesting or critical situation in the training loss or some other training metric one can quickly ask the training to save the checkpoint of interest and be able to reproduce the current situation at will.

The main use of this feature is around observing training loss spikes and divergences.


## Prevention

The easiest way to avoid losing training time is to prevent some types of problems from happening. While one can't prevent a GPU from failing, other than ensuring that adequate cooling is provided, one can certain ensure that there is enough of disc space remaining for the next few days of training. This is typically done by running scheduled watchdogs that monitor various resources and alert the operator of possible problems long before they occur.

### Scheduled Watchdogs

Before we discuss the various watchdogs it's critical that you have a mechanism that allows you to run scheduled jobs. In the Unix world this is implemented by the [`crontab` facility](https://en.wikipedia.org/wiki/Cron).

Here is an example of a program that will be launched every hour:

```
0 * * * * /path/to/watch-fs.sh
```
The link above explains how to configure a crontab job to run at various other frequencies.

To setup a crontab, execute `crontab -e` and check which jobs are scheduled `crontab -l`.

The reason I don't go into many details is because many SLURM environments don't provide access to the `crontab` facility. And therefore one needs to use other approaches to scheduling jobs.

The section on [Crontab Emulation](../slurm#crontab-emulation) discusses how to implement crontab-like slurm emulation and also
[Self-perpetuating SLURM jobs](../slurm#self-perpetuating-slurm-jobs).


### Notification facility

Then you need to have one or more notification facilities.

The simplest one is to use an email where the alerts will be sent. To make this one work you need to ensure that you have a way to send an email from the slurm job. If it isn't already available you can contact your sysadmin or alternatively you might be able to use an external SMTP server provider.

In addition to email you could probably also setup a pager-like SMS alerting and if you use Slack to send slack-notifications to a channel of your choice.

Once you understand how to schedule watchdogs and you have a notification facility working let's next discuss the critical watchdogs.

### Job running watchdog

The most obvious watchdog is one which checks that there is a training SLURM job running and optionally that there is queue of jobs scheduled to run.

Here is an example [slurm-status.py](./slurm-status.py) that was used during BLOOM-176B training. This watchdog was sending an email if a job was detected to be neither running nor scheduled and it was also pushing its checks into the main training's log file. As we used [Crontab Emulation](../slurm#crontab-emulation), we simply needed to drop  [slurm-status.slurm](./slurm-status.slurm) into the `cron/cron.hourly/` folder and the previously launched SLURM crontab emulating scheduler would run it approximately once an hour.

The key part of the slurm job is:
```
tools/slurm-status.py --job-name $WATCH_SLURM_NAME 2>&1 | tee -a $MAIN_LOG_FILE
```
which tells the script which job name to watch for, and you can also see that it logs into a log file.

You can now adapt these scripts to your needs with minimal changes of editing the path and email addresses.

### Low Disc Space Alerts

The next biggest issue is running out of disc space. If your checkpoints are large and are saved frequently and aren't offloaded elsewhere it's easy to quickly run out of space. Moreover, typically multiple team members share the same cluster and it could be that your colleagues could quickly consume a lot of disc space. Ideally, you'd have a storage partition that is dedicated to your training only, but often this is difficult to accomplish. In either case you need to know when disc space is low and space making action is to be performed.

Now what should be the threshold at which the alerts are made. They need to be made not too soon as people will start ignoring your alerts if you start sending those at 50% usage. But also the percentage isn't always applicable, because if you have a huge disc space shared with others, 5% of that disc space could amount in many TBs of free disc space. Therefore really you should know how often you write your checkpoints and know how many TBs of disc space you need daily.

Use case: During BLOOM training we were writing 329GB checkpoints every 3 hours, therefore we were consuming 2.6TB a day!

Moreover, often there will be multiple partitions - faster IO partitions dedicated to checkpoint writing, and slower partitions dedicated to code and libraries, and possibly various other partitions that could be in use and all of those need to be monitored if their availability is required for the training not crashing.

Here is another gotcha - when it comes to distributed file systems not all filesystems can reliably give you a 100% of disc space you rented or purchased. In fact with some of those types you can only reliably use about 80% of the allocated storage space. The problem is that these systems use physical discs that they re-balance at the scheduled periods or triggered events, and thus any of these discs could reach 100% of capacity and lead to a failed write, which would crash a training process, even though `df` would report only 80% space usage on the partition. We didn't have this problem while training BLOOM-176B, but we had it when we trained IDEFICS-80B - 80% there was the new 100%. How do you know if you have this issue - well, usually you discover it while you prepare for the training.

And this is not all. There is another issue of inodes availability and some storage partitions don't have very large quotas. Python packages are notorious for having hundreds to thousands of small files, which take little space, but add up to tens of thousands and suddenly you have TBs of free disc space available, but no free inodes and that would lead to training crashing.

Finally, many distributed partitions don't show you the usage stats in real time and could take up to 30min to update.

footnote: Use `df -ih` to see the inodes quota and the current usage.

So here is [fs-watchdog.py](./fs-watchdog.py) that was used during BLOOM-176B training. This watchdog was sending an email if any of the storage requirements thresholds haven't been met and here is the corresponding [fs-watchdog.slurm](./fs-watchdog.slurm) that was driving it.

If you study the watchdog code you can see that for each partition we were monitoring both the disc usage and inodes. We used special quota tools provided by the HPC to get instant stats for some partitions, but these tools didn't work for all partitions and there we had to fallback to using `df` and even a much slower `du`. So it should be easy to adapt to your usecase.


### Dealing with slow memory leaks

Some programs develop tiny memory leaks which can be very difficult to debug. Do not confuse those with the usage of MMAP where the program uses the CPU memory to read quickly read data from and where the memory usage could appear to grow over time, but this is not real. You can [A Deep Investigation into MMAP Not Leaking Memory](https://stasosphere.com/entrepreneur-being/301-mmap-memory-leak-investigation/) to understand why.

Of course, ideally one would analyse their software and fix the leak, but at times the leak could be coming from a 3rd party package or can be very difficult to diagnose and there isn't often a time to do that.

When it comes to GPU memory, there is the possible issue of memory fragmentation, where over time more and more tiny unused memory segments add up and make the GPU appear to have a good amount of free memory, but when the program tries to allocate this memory it fails with the OOM error like:

```
RuntimeError: CUDA out of memory. Tried to allocate 304.00 MiB (GPU 0; 8.00 GiB total capacity; 142.76 MiB already allocated; 6.32 GiB free; 158.00 MiB reserved in total by PyTorch)
```
In this example if there are 6.32GB free, how comes that 304MB couldn't be allocated.

One of the approaches my team developed during IDEFICS-80B training where there was some tiny CPU memory leak that would often take multiple days to lead to running out of CPU memory was to install a watchdog inside the training loop that would check the memory usage and if a threshold was reached it'd voluntarily exit the training loop. The next training job will resume with all the CPU memory reclaimed.

footnote: The reality of machine learning trainings is that not all problems can be fixed with limited resources and often times a solid workaround provides for a quicker finish line, as compared to "stopping the presses" and potentially delaying the training by weeks while trying to figure out the problem. For example we trained BLOOM-176B with `CUDA_LAUNCH_BLOCKING=1` because the training would hang without it and after multiple failed attempts to diagnose that we couldn't afford any more waiting and had to proceed as is. Luckily this environment variable that normally is used for debug purposes and which in theory should make everything slower didn't actually make any difference to our throughput. But we have never figured out what the problem was and today it doesn't matter at all that we didn't, as we moved on with other projects which aren't impacted by that issue.

The idea is similar to the kill switch and save switch discussed earlier, but here instead of polling for a specific file appearance we simply watch how much resident memory is used:

```
import psutil
for batch in iterator:
    total_used_percent = psutil.virtual_memory().percent
    if total_used_percent > 0.95:
        print(f"Exiting early since the cpu memory is almost full: ({total_used_percent}%)")
        save_checkpoint()
        sys.exit()
```

Similar heuristics could be used for setting a threshold for GPU memory usage, except one needs to be aware of caching, so to get the actual memory usage you'd need to do first run the garbage collector then empty the cache and only the you will get real memory usage stats:

```
import gc
import torch

for batch in iterator:
    gc.collect()
    torch.cuda.empty_cache()

    # get mem usage in GBs and exit if less than 2GB of free gpu memory remain
    free, total = map(lambda x: x/2**30, torch.cuda.mem_get_info());
    if free < 2:
        print(f"Exiting early since the gpu memory is almost full: ({free}GB remain)")
        save_checkpoint()
        sys.exit()
```

footnote: don't do this unless you really have to, since caching makes things faster. Ideally figure out the fragmentation issue instead. For example, look up [`max_split_size_mb`](https://pytorch.org/docs/stable/notes/cuda.html#environment-variables) that controls how memory is allocated.


###