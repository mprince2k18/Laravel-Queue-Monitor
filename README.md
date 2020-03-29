# Laravel Queue Monitor

[![Latest Stable Version](https://img.shields.io/packagist/v/romanzipp/laravel-queue-monitor.svg?style=flat-square)](https://packagist.org/packages/romanzipp/laravel-queue-monitor)
[![Total Downloads](https://img.shields.io/packagist/dt/romanzipp/laravel-queue-monitor.svg?style=flat-square)](https://packagist.org/packages/romanzipp/laravel-queue-monitor)
[![License](https://img.shields.io/packagist/l/romanzipp/laravel-queue-monitor.svg?style=flat-square)](https://packagist.org/packages/romanzipp/laravel-queue-monitor)
[![Code Quality](https://img.shields.io/scrutinizer/g/romanzipp/laravel-queue-monitor.svg?style=flat-square)](https://scrutinizer-ci.com/g/romanzipp/laravel-queue-monitor/?branch=master)
[![Build Status](https://img.shields.io/travis/romanzipp/Laravel-Queue-Monitor.svg?style=flat-square)](https://travis-ci.org/romanzipp/Laravel-Queue-Monitor)

This package offers monitoring like [Laravel Horizon](https://laravel.com/docs/horizon) for database queue.

## Features

- Monitor jobs like [Laravel Horizon](https://laravel.com/docs/horizon) for any queue
- Handle failing jobs with storing exception
- Monitor job progress
- Get an estimated time remaining for a job
- Store additional data for a job monitoring

## Installation

```
composer require romanzipp/laravel-queue-monitor
```

**If you use Laravel 5.5+ you are already done, otherwise continue.**

Add Service Provider to your app.php configuration file:

```php
romanzipp\QueueMonitor\Providers\QueueMonitorProvider::class,
```

## Configuration

Copy configuration & migration to your project:

```
$ php artisan vendor:publish --provider="romanzipp\QueueMonitor\Providers\QueueMonitorProvider"
```

Migrate the Queue Monitoring table. The table name can be configured in the config file or via the published migration.

```
$ php artisan migrate
```

## Usage

To monitor a job, simply add the `romanzipp\QueueMonitor\Traits\IsMonitored` Trait.

```php
use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use romanzipp\QueueMonitor\Traits\IsMonitored; // <---

class ExampleJob implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;
    use IsMonitored; // <---
}
```

### Set progress

You can set a **progress value** (0-100) to get an estimation of a job progression.

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use romanzipp\QueueMonitor\Traits\IsMonitored;

class ExampleJob implements ShouldQueue
{
    use IsMonitored;

    public function handle()
    {
        $this->queueProgress(0);

        // Do something...

        $this->queueProgress(50);

        // Do something...

        $this->queueProgress(100);
    }
}
``` 

### Set progress in chunk

A common scenario for a job is iterating through large collections.

This example job loops through a large amount of users and updates it's progress value with each chunk iteration.

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Database\Eloquent\Collection;
use romanzipp\QueueMonitor\Traits\IsMonitored;

class ExampleJob implements ShouldQueue
{
    use IsMonitored;

    public function handle()
    {
        $usersCount = User::count();

        $perChunk = 50;

        User::query()
            ->chunk($perChunk, function (Collection $users) use ($perChunk, $usersCount) {

                $this->queueProgressChunk($usersCount‚ $perChunk);

                foreach ($users as $user) {
                    // ...
                }
            });
    }
}
``` 

### Set custom data

This package also allows to set custom data on the monitoring model.

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use romanzipp\QueueMonitor\Traits\IsMonitored;

class ExampleJob implements ShouldQueue
{
    use IsMonitored;

    public function handle()
    {
        $this->queueData(['foo' => 'Bar']);
        
        // WARNING! This is overriding the monitoring data
        $this->queueData(['bar' => 'Foo']);

        // To preserve previous data and merge the given payload, set the $merge parameter true
        $this->queueData(['bar' => 'Foo'], true);
    }
}
``` 

### Retrieve processed Jobs

```php
use romanzipp\QueueMonitor\Models\Monitor;

$job = Monitor::query()->first();

// Check the current state of a job
$job->isFinished();
$job->hasFailed();
$job->hasSucceeded();

// Exact start & finish dates with milliseconds
$job->getStartedAtExact();
$job->getFinishedAtExact();

// If the job is still running, get the estimated seconds remaining
// Notice: This requires a progress to be set
$job->getRemainingSeconds();

// Retrieve any data that has been set while execution
$job->getData();

// Get the base name of the executed job
$job->getBasename();
```

### Model Scopes

```php
use romanzipp\QueueMonitor\Models\Monitor;

// Filter by Status
Monitor::failed();
Monitor::succeeded();

// Filter by Date
Monitor::lastHour();
Monitor::today();

// Chain Scopes
Monitor::today()->failed();
```

## ToDo

- [ ] Offer configuration to use own monitoring model
- [ ] Add Job & Artisan Command for automatic cleanup of old database entries

## Upgrading from 1.0 to 2.0

### Database

Add the following fields to your queue monitor table:

- `exception_message` (string, nullable)
- `exception_class` (string, nullable)

### The Job Trait

The job trait has been renamed to a more intuitive name.

```diff
- use romanzipp\QueueMonitor\Traits\QueueMonitor;
+ use romanzipp\QueueMonitor\Traits\IsMonitored; 
```

### The Monitor Model

#### Changed Methods

```diff
- $monitor->basename()
- $monitor->basename
+ $monitor->getBaseame()
```

```diff
- $monitor->parsed_data
+ $monitor->getData()
```

```diff
- $monitor->remaing_seconds
+ $monitor->getRemainingSeconds()
```

```diff
- $monitor->startedAtExact()
+ $monitor->getStartedAtExact()
```

```diff
- $monitor->finishedAtExact()
+ $monitor->getFinishedAtExact()
```

```diff
- $monitor->isSucceeded()
+ $monitor->hasSucceeded()
```

The `getRemainingSeconds()` method now always returns a `float` instead of `float|null`,

```diff
- public function getRemainingSeconds(): ?float
+ public function getRemainingSeconds(): float
```

#### New Methods

```diff
+ $monitor->hasFailed()
```

----

This package was inspired by gilbitron's [laravel-queue-monitor](https://github.com/gilbitron/laravel-queue-monitor) which is not maintained anymore.
