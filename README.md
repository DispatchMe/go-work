# DispatchMe/go-work [![GoDoc](https://godoc.org/github.com/DispatchMe/go-work?status.png)](https://godoc.org/github.com/DispatchMe/go-work)

**This is a fork of [gocraft/work](https://www.github.com/gocraft/work) that removes their strange usage of custom contexts. The use of reflection to support that was detracting from efficiency and didn't feel very "go-like", so we made this change**

DispatchMe/go-work lets you enqueue and processes background jobs in Go. Jobs are durable and backed by Redis. Very similar to Sidekiq for Go.

* Fast and efficient. Faster than [this](https://www.github.com/jrallison/go-workers), [this](https://www.github.com/benmanns/goworker), and [this](https://www.github.com/albrow/jobs). See below for benchmarks.
* Reliable - don't lose jobs even if your process crashes.
* Middleware on jobs -- good for metrics instrumentation, logging, etc.
* If a job fails, it will be retried a specified number of times.
* Schedule jobs to happen in the future.
* Enqueue unique jobs so that only one job with a given name/arguments exists in the queue at once.
* Web UI to manage failed jobs and observe the system.
* Periodically enqueue jobs on a cron-like schedule.

## Run tests

Redis must be installed to avoid a panic when running tests.

## Enqueue new jobs

To enqueue jobs, you need to make an Enqueuer with a redis namespace and a redigo pool. Each enqueued job has a name and can take optional arguments. Arguments are k/v pairs (serialized as JSON internally).

```go
package main

import (
	"github.com/gomodule/redigo/redis"
	"github.com/DispatchMe/go-work"
)

// Make a redis pool
var redisPool = &redis.Pool{
	MaxActive: 5,
	MaxIdle: 5,
	Wait: true,
	Dial: func() (redis.Conn, error) {
		return redis.Dial("tcp", ":6379")
	},
}

type SendEmailJobParameters struct {
	Address string
	Subject string
	CustomerID int
}

// Make an enqueuer with a particular namespace
var enqueuer = work.NewEnqueuer("my_app_namespace", redisPool)

func main() {
	// Enqueue a job named "send_email" with the specified parameters.
	_, err := enqueuer.Enqueue("send_email", &SendEmailJobParameters{Address: "test@example.com", Subject: "hello world", CustomerID: 4})
	if err != nil {
		log.Fatal(err)
	}
}


```

## Process jobs

In order to process jobs, you'll need to make a WorkerPool. Add middleware and jobs to the pool, and start the pool.

```go
package main

import (
	"github.com/gomodule/redigo/redis"
	"github.com/DispatchMe/go-work"
	"os"
	"os/signal"
)

// Make a redis pool
var redisPool = &redis.Pool{
	MaxActive: 5,
	MaxIdle: 5,
	Wait: true,
	Dial: func() (redis.Conn, error) {
		return redis.Dial("tcp", ":6379")
	},
}

func main() {
	// Make a new pool. Arguments:
	// 10 is the max concurrency
	// "my_app_namespace" is the Redis namespace
	// redisPool is a Redis pool
	pool := work.NewWorkerPool(10, "my_app_namespace", redisPool)

	// Add middleware that will be executed for each job
	pool.Middleware(LogMiddleware)

	// Map the name of jobs to handler functions
	pool.Job("send_email", SendEmailHandler)

	// Customize options:
	pool.JobWithOptions("export", JobOptions{Priority: 10, MaxFails: 1}, ExportHandler)

	// Start processing jobs
	pool.Start()

	// Wait for a signal to quit:
	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, os.Interrupt, os.Kill)
	<-signalChan

	// Stop the pool
	pool.Stop()
}

func LogMiddleware(ctx *work.Context, next work.NextMiddlewareFunc) error {
	fmt.Println("Starting job: ", ctx.Job.Name)
	return next()
}

func SendEmailHandler(ctx *work.Context) error {
	// Extract arguments:
	args := new(SendEmailJobParameters)
	err := ctx.Job.UnmarshalPayload(args)
	if err != nil {
		return err
	}

	// Go ahead and send the email...
	// sendEmailTo(args.Address, args.Subject)

	return nil
}

func ExportHandler(ctx *work.Context) error {
	return nil
}
```

## Special Features

### Check-ins

Since this is a background job processing library, it's fairly common to have jobs that that take a long time to execute. Imagine you have a job that takes an hour to run. It can often be frustrating to know if it's hung, or about to finish, or if it has 30 more minutes to go.

To solve this, you can instrument your jobs to "checkin" every so often with a string message. This checkin status will show up in the web UI. For instance, your job could look like this:

```go
func ExportHandler(ctx *work.Context) error {
	rowsToExport := getRows()
	for i, row := range rowsToExport {
		exportRow(row)
		if i % 1000 == 0 {
			ctx.Job.Checkin("i=" + fmt.Sprint(i))   // Here's the magic! This tells DispatchMe/go-work our status
		}
	}
}

```

Then in the web UI, you'll see the status of the worker:

| Name | Arguments | Started At | Check-in At | Check-in |
| --- | --- | --- | --- | --- |
| export | {"account_id": 123} | 2016/07/09 04:16:51 | 2016/07/09 05:03:13 | i=335000 |

### Scheduled Jobs

You can schedule jobs to be executed in the future. To do so, make a new ```Enqueuer``` and call its ```EnqueueIn``` method:

```go
enqueuer := work.NewEnqueuer("my_app_namespace", redisPool)
secondsInTheFuture := 300
_, err := enqueuer.EnqueueIn("send_welcome_email", secondsInTheFuture, work.Q{"address": "test@example.com"})
```

### Unique Jobs

You can enqueue unique jobs so that only one job with a given name/arguments exists in the queue at once. For instance, you might have a worker that expires the cache of an object. It doesn't make sense for multiple such jobs to exist at once. Also note that unique jobs are supported for normal enqueues as well as scheduled enqueues.

```go
enqueuer := work.NewEnqueuer("my_app_namespace", redisPool)
job, err := enqueuer.EnqueueUnique("clear_cache", work.Q{"object_id_": "123"}) // job returned
job, err = enqueuer.EnqueueUnique("clear_cache", work.Q{"object_id_": "123"}) // job == nil -- this duplicate job isn't enqueued.
job, err = enqueuer.EnqueueUniqueIn("clear_cache", 300, work.Q{"object_id_": "789"}) // job != nil (diff id)
```

### Periodic Enqueueing (Cron)

You can periodically enqueue jobs on your DispatchMe/go-work cluster using your worker pool. The [scheduling specification](https://godoc.org/github.com/robfig/cron#hdr-CRON_Expression_Format) uses a Cron syntax where the fields represent seconds, minutes, hours, day of the month, month, and week of the day, respectively. Even if you have multiple worker pools on different machines, they'll all coordinate and only enqueue your job once.

```go
pool := work.NewWorkerPool(10, "my_app_namespace", redisPool)
pool.PeriodicallyEnqueue("0 0 * * * *", "calculate_caches") // This will enqueue a "calculate_caches" job every hour
pool.Job("calculate_caches", CalculateCaches) // Still need to register a handler for this job separately
```

## Run the Web UI

The web UI provides a view to view the state of your DispatchMe/go-work cluster, inspect queued jobs, and retry or delete dead jobs.

Building an installing the binary:
```bash
go get github.com/DispatchMe/go-work/cmd/workwebui
go install github.com/DispatchMe/go-work/cmd/workwebui
```

Then, you can run it:
```bash
workwebui -redis=":6379" -ns="work" -listen=":5040"
```

Navigate to ```http://localhost:5040/```.

You'll see a view that looks like this:

![Web UI Screenshot](https://gocraft.github.io/work/images/webui.png)

## Design and concepts

### Enqueueing jobs

* When jobs are enqueued, they're serialized with JSON and added to a simple Redis list with LPUSH.
* Jobs are added to a list with the same name as the job. Each job name gets its own queue. Whereas with other job systems you have to design which jobs go on which queues, there's no need for that here.

### Scheduling algorithm

* Each job lives in a list-based queue with the same name as the job.
* Each of these queues can have an associated priority. The priority is a number from 1 to 100000.
* Each time a worker pulls a job, it needs to choose a queue. It chooses a queue probabilistically based on its relative priority.
* If the sum of priorities among all queues is 1000, and one queue has priority 100, jobs will be pulled from that queue 10% of the time.
* Obviously if a queue is empty, it won't be considered.
* The semantics of "always process X jobs before Y jobs" can be accurately approximated by giving X a large number (like 10000) and Y a small number (like 1).

### Processing a job

* To process a job, a worker will execute a Lua script to atomically move a job its queue to an in-progress queue.
* The worker will then run the job. The job will either finish successfully or result in an error or panic.
  * If the process completely crashes, the reaper will eventually find it in its in-progress queue and requeue it.
* If the job is successful, we'll simply remove the job from the in-progress queue.
* If the job returns an error or panic, we'll see how many retries a job has left. If it doesn't have any, we'll move it to the dead queue. If it has retries left, we'll consume a retry and add the job to the retry queue.

### Workers and WorkerPools

* WorkerPools provide the public API of DispatchMe/go-work.
  * You can attach jobs and middleware to them.
  * You can start and stop them.
  * Based on their concurrency setting, they'll spin up N worker goroutines.
* Each worker is run in a goroutine. It will get a job from redis, run it, get the next job, etc.
  * Each worker is independent. They are not dispatched work -- they get their own work.

### Retry job, scheduled jobs, and the requeuer

* In addition to the normal list-based queues that normal jobs live in, there are two other types of queues: the retry queue and the scheduled job queue.
* Both of these are implemented as Redis z-sets. The score is the unix timestamp when the job should be run. The value is the bytes of the job.
* The requeuer will occasionally look for jobs in these queues that should be run now. If they should be, they'll be atomically moved to the normal list-based queue and eventually processed.

### Dead jobs

* After a job has failed a specified number of times, it will be added to the dead job queue.
* The dead job queue is just a Redis z-set. The score is the timestamp it failed and the value is the job.
* To retry failed jobs, use the UI or the Client API.

### The reaper

* If a process crashes hard (eg, the power on the server turns off or the kernal freezes), some jobs may be in progress and we won't want to lose them. They're safe in their in-progress queue.
* The reaper will look for worker pools without a heartbeat. It will scan their in-progress queues and requeue anything it finds.

### Unique jobs

* You can enqueue unique jobs such that a given name/arguments are on the queue at once.
* Both normal queues and the scheduled queue are considered.
* When a unique job is enqueued, we'll atomically set a redis key that includes the job name and arguments and enqueue the job.
* When the job is processed, we'll delete that key to permit another job to be enqueued.

### Periodic Jobs

* You can tell a worker pool to enqueue jobs periodically using a cron schedule.
* Each worker pool will wake up every 2 minutes, and if jobs haven't been scheduled yet, it will schedule all the jobs that would be executed in the next five minutes.
* Each periodic job that runs at a given time has a predictable byte pattern. Since jobs are scheduled on the scheduled job queue (a Redis z-set), if the same job is scheduled twice for a given time, it can only exist in the z-set once.

### Terminology reference
* "worker pool" - a pool of workers
* "worker" - an individual worker in a single goroutine. Gets a job from redis, does job, gets next job...
* "heartbeater" or "worker pool heartbeater" - goroutine owned by worker pool that runs concurrently with workers. Writes the worker pool's config/status (aka "heartbeat") every 5 seconds.
* "heartbeat" - the status written by the heartbeater.
* "observer" or "worker observer" - observes a worker. Writes stats. makes "observations".
* "worker observation" - A snapshot made by an observer of what a worker is working on.
* "periodic enqueuer" - A process that runs with a worker pool that periodically enqueues new jobs based on cron schedules.
* "job" - the actual bundle of data that constitutes one job
* "job name" - each job has a name, like "create_watch"
* "job type" - backend/private nomenclature for the handler+options for processing a job
* "queue" - each job creates a queue with the same name as the job. only jobs named X go into the X queue.
* "retry jobs" - If a job fails and needs to be retried, it will be put on this queue.
* "scheduled jobs" - Jobs enqueued to be run in th future will be put on a scheduled job queue.
* "dead jobs" - If a job exceeds its MaxFails count, it will be put on the dead job queue.


## Benchmarks

The benches folder used to contain various benchmark code. In each case, we enqueued 100k jobs across 5 queues. The jobs were almost no-op jobs: they simply incremented an atomic counter. We then measured the rate of change of the counter to obtain our measurement. These were some test results:

| Library | Speed |
| --- | --- |
| [DispatchMe/go-work](https://www.github.com/DispatchMe/go-work) | **20944 jobs/s** |
| [jrallison/go-workers](https://www.github.com/jrallison/go-workers) | 19945 jobs/s |
| [benmanns/goworker](https://www.github.com/benmanns/goworker) | 10328.5 jobs/s |
| [albrow/jobs](https://www.github.com/albrow/jobs) | 40 jobs/s |

The comparison benchmarks were run against repositories that were stale and unmaintained by fall of 2018. Invalid
import paths were causing tests to fail in go-lib, which has background and indexer packages that rely on this
repository.  As the benchmarks were no longer needed, they were removed.

## Authors

* Jonathan Novak -- [https://github.com/cypriss](https://github.com/cypriss)
* Tai-Lin Chu -- [https://github.com/taylorchu](https://github.com/taylorchu)
* Tyler Smith -- [https://github.com/tyler-smith](https://github.com/tyler-smith)
* Sponsored by [UserVoice](https://eng.uservoice.com)
