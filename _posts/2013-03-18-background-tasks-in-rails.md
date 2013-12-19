---
layout: post
---

_Update: I tried to install cronic in my Rails app and it failed because cronic requires Rails 3.2.13+. I'm now investigating daemon-kit._

The problem addressed in this post has to do with managing background jobs in an existing rails 2.3.x app. The focus of this article is decidedly NOT how we got to this point but how can we fix it now. The app has background processing in the form of a homegrown queue mechanism ("queue") and 100+ cron jobs ("cron"). I try to document our current problems and my thought process and reasoning as I consider the options available.

## The Problem(s)

Currently our background processes (queues and crons) run on a separate server from the web application but this isn't the problem. In fact this may be the best feature of our background processing system (only half joking).

### Everything Runs Through Cron

Everything runs through cron. Even the queue is processed through cron:

    */5 * * * * rake_runner app background:jobs
    1,6,11,16,21,26,31,36,41,46,51,56 * * * * rake_runner app email:process

This is just a sample of the 100+ jobs we have running. It illustrates both kinds of jobs: queued and traditional cron jobs.

The "background:jobs" cron task is what handles our background tasks. As things happen in the web app background tasks get added to the background:jobs queue. Every 5 minutes this process checks the queue and processes the jobs that have not yet been run. In order to manage potential **overlap** there is a hard-coded limit to the number of tasks that can be processed at a time. This hard-coded limit needs fixing.

Since most of the cron jobs run rake tasks each one fires up its own instance of the rails app every time it runs. We have tried to schedule things so that they wouldn't **overlap** and impact each other but there are no guarantees. We need better control over this.

### No Exception Handling

Right now we have NO **exception handling** in our background processes. We are logging pretty heavily but we only view the logs when there is a problem detected. We've had instances where a background process failed and we only found out about it because the customer noticed that some expected action hadn't happened. If there are problems in the background processes we need to know about it before a user notices and tells us. This needs fixing.

Just to be clear, cron is NOT the problem. The way we use cron is the problem. We need more.

### Cron is Disconnected From Code

Any time we need to view or make a change to the background processes (which isn't very often) we have to login to the server. Logging in to the server is not a big deal. We have the opportunity to update our cron server (part of the impetus for this whole thought process) and realized that our cron jobs are not connected with anything else in the app. Sure the cron jobs run rake tasks but cron is not controlled by, configured by, deployed by, or managed by anything that comes from our codebase. Everything is a manual effort. If the cron server were to die right now while I'm typing it would be a major catastrophe. (Just a second, I'm going to go backup the crontab.) This needs to be fixed. Our background processes need to be **connected to our codebase**.

### Summary of Problems

- Potential overlap
- Not integrated with our exception handling solution ([Airbrake](airbrake.io)).
- Disconnected from our codebase

## Requirements and Constraints

These are the requirements/constraints for our solution:

- It must handle queues AND crons.
- It must run separately from the web app.
- It must prevent/handle overlap.
- It must handle exceptions or integrate with our existing solution (Airbrake).
- It must be managed from within our codebase.
- It must run all jobs from within a single instance.

## The Contenders

I considered the following possibilities.

- current solution (do nothing)
- "[whenever](https://github.com/javan/whenever)" gem
- "[delayed_job](https://github.com/collectiveidea/delayed_job)" gem
- "[resque](https://github.com/defunkt/resque)" solution
- "[rufus-scheduler](https://github.com/jmettraux/rufus-scheduler)" gem
- "custom daemon" solution
- "[cronic](https://github.com/jkraemer/cronic)" gem

### Current Solution

Here is how our current solution matches up with our requirements/constraints.

<table class="table table-striped">
  <tr>
    <th>Solution</th>
    <th>Queues and Crons</th>
    <th>Runs Separately</th>
    <th>Manages Overlap</th>
    <th>Handles Exceptions</th>
    <th>Within Codebase</th>
    <th>Single Instance</th>
  </tr>
  <tr>
    <td>Current</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>No</td>
    <td>No</td>
    <td>No</td>
    <td>No</td>
  </tr>
</table>

As described above our current solution has a few problems: overlap, exceptions, outside of codebase, and it runs multiple instances.

### Whenever Gem

The whenever gem looks like a wonderful single-purpose tool for writing and deploying cron jobs. In the table below we see how it matches up with our requirements.

<table class="table table-striped">
  <tr>
    <th>Solution</th>
    <th>Queues and Crons</th>
    <th>Runs Separately</th>
    <th>Manages Overlap</th>
    <th>Handles Exceptions</th>
    <th>Within Codebase</th>
    <th>Single Instance</th>
  </tr>
  <tr>
    <td>Whenever</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>No</td>
    <td>No</td>
    <td>Yes</td>
    <td>No</td>
  </tr>
</table>

The only improvement that whenever gives us is that the cron jobs would be managed within our existing codebase. This would be an incremental improvement over what we have now but not enough.

### Delayed Job

Delayed job is a database-based priority queuing system with workers. I've never worked with it but here is how I think it matches up with our requirements.

<table class="table table-striped">
  <tr>
    <th>Solution</th>
    <th>Queues and Crons</th>
    <th>Runs Separately</th>
    <th>Manages Overlap</th>
    <th>Handles Exceptions</th>
    <th>Within Codebase</th>
    <th>Single Instance</th>
  </tr>
  <tr>
    <td>Delayed Job</td>
    <td>No</td>
    <td>Yes</td>
    <td>No</td>
    <td>No</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
</table>

Using Delayed Job would meet the queues part of our requirement but not the crons part and would effectively replace our current working background jobs solution. In order to use Delayed Job to meet our crons needs would require a separate process to schedule the placement of jobs on the Delayed Job queue. Also, Delayed Job would require a separate process to manager the worker(s) that process the jobs. By my counts that is an extra process beyond what we have now. We would have to manage the overlap and exceptions ourselves.

### Resque

<table class="table table-striped">
  <tr>
    <th>Solution</th>
    <th>Queues and Crons</th>
    <th>Runs Separately</th>
    <th>Manages Overlap</th>
    <th>Handles Exceptions</th>
    <th>Within Codebase</th>
    <th>Single Instance</th>
  </tr>
  <tr>
    <td>Resque</td>
    <td>No</td>
    <td>Yes</td>
    <td>No</td>
    <td>No</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
</table>

I think the evaluation of Resque is similar to that for Delayed Job except for it adds moving parts. Out of the box it uses Redis for the queue. So, in addition to the workers there is now another database.

### Rufus Scheduler Gem

Rufus Scheduler is a gem for scheduling pieces of code to run AT a certain time, IN a certain time, or every X time, or using a cron-like schedule. It looks promising on paper.

<table class="table table-striped">
  <tr>
    <th>Solution</th>
    <th>Queues and Crons</th>
    <th>Runs Separately</th>
    <th>Manages Overlap</th>
    <th>Handles Exceptions</th>
    <th>Within Codebase</th>
    <th>Single Instance</th>
  </tr>
  <tr>
    <td>Rufus Scheduler</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
</table>

So everything looks good so far. Running separately would require some kind of daemon to manage. It has built-in configurable overlap prevention. Rufus requires manual exception configuration and provides working examples. Rufus is looking pretty good.

### Custom Daemon

<table class="table table-striped">
  <tr>
    <th>Solution</th>
    <th>Queues and Crons</th>
    <th>Runs Separately</th>
    <th>Manages Overlap</th>
    <th>Handles Exceptions</th>
    <th>Within Codebase</th>
    <th>Single Instance</th>
  </tr>
  <tr>
    <td>Custom Daemon</td>
    <td>Yes *</td>
    <td>Yes *</td>
    <td>Yes *</td>
    <td>Yes *</td>
    <td>Yes *</td>
    <td>Yes *</td>
  </tr>
</table>

\* There is a big caviat to each of these "Yes" responses and that is "Yes, WITH DEVELOPMENT." Just about anything is possible if we want to create a custom daemon. We have no desire or time to create our own custom daemon.

### Cronic

Cronic is a cron-like scheduler daemon for Rails. Complete with Rufus integration (see above), startup and shutdown scripts, capistrano scripts, and exception handling with Airbrake.

<table class="table table-striped">
  <tr>
    <th>Solution</th>
    <th>Queues and Crons</th>
    <th>Runs Separately</th>
    <th>Manages Overlap</th>
    <th>Handles Exceptions</th>
    <th>Within Codebase</th>
    <th>Single Instance</th>
  </tr>
  <tr>
    <td>Cronic Gem</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
</table>

Cronic looks just about perfect for our needs right out of the box. It supports all of the scheduling features of Rufus Scheduler, integrates with our configured exception handling, and provides its own deploy, startup, shutdown, and restart scripts. For our needs at this time I think Cronic is the best option.

### Summary

<table class="table table-striped">
  <tr>
    <th>Solution</th>
    <th>Queues and Crons</th>
    <th>Runs Separately</th>
    <th>Manages Overlap</th>
    <th>Handles Exceptions</th>
    <th>Within Codebase</th>
    <th>Single Instance</th>
  </tr>
  <tr>
    <td>Current</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>No</td>
    <td>No</td>
    <td>No</td>
    <td>No</td>
  </tr>
  <tr>
    <td>Whenever</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>No</td>
    <td>No</td>
    <td>Yes</td>
    <td>No</td>
  </tr>
  <tr>
    <td>Delayed Job</td>
    <td>No</td>
    <td>Yes</td>
    <td>No</td>
    <td>No</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
  <tr>
    <td>Resque</td>
    <td>No</td>
    <td>Yes</td>
    <td>No</td>
    <td>No</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
  <tr>
    <td>Rufus Scheduler</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
  <tr>
    <td>Custom Daemon</td>
    <td>Yes *</td>
    <td>Yes *</td>
    <td>Yes *</td>
    <td>Yes *</td>
    <td>Yes *</td>
    <td>Yes *</td>
  </tr>
  <tr>
    <td>Cronic Gem</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
</table>

\* Yes, with development effort.

I didn't intend to stack the deck for Cronic but in the end the analysis made it seem that way. Hopefully this was helpful. I can't wait to get started with Cronic.

I didn't intentionally misrepresent anything. I you see something that isn't right let me know.

<a href="https://github.com/richcorbs/Feedback/issues/new" class="btn btn-mini btn-info">Feedback</a>
