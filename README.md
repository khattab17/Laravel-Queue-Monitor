# Laravel Queue Monitor

[![Latest Stable Version](https://img.shields.io/packagist/v/romanzipp/laravel-queue-monitor.svg?style=flat-square)](https://packagist.org/packages/romanzipp/laravel-queue-monitor)
[![Total Downloads](https://img.shields.io/packagist/dt/romanzipp/laravel-queue-monitor.svg?style=flat-square)](https://packagist.org/packages/romanzipp/laravel-queue-monitor)
[![License](https://img.shields.io/packagist/l/romanzipp/laravel-queue-monitor.svg?style=flat-square)](https://packagist.org/packages/romanzipp/laravel-queue-monitor)
[![GitHub Build Status](https://img.shields.io/github/actions/workflow/status/romanzipp/Laravel-Queue-Monitor/tests.yml?branch=master&label=tests&style=flat-square)](https://github.com/romanzipp/Laravel-Queue-Monitor/actions)

This package offers monitoring like [Laravel Horizon](https://laravel.com/docs/horizon) for database queue.

## Features

- Monitor jobs like [Laravel Horizon](https://laravel.com/docs/horizon) for any queue
- Handle failing jobs with storing exception
- Monitor job progress
- Get an estimated time remaining for a job
- Store additional data for a job monitoring
- Retry jobs via the UI

## Installation

✨ **See [Upgrade Guide](https://github.com/romanzipp/Laravel-Queue-Monitor/releases/tag/4.0.0) if you are updating to 4.0** ✨

```
composer require romanzipp/laravel-queue-monitor
```

See [romanzipp/Laravel-Queue-Monitor-Nova](https://github.com/romanzipp/Laravel-Queue-Monitor-Nova) for **Laravel Nova** resources & metrics.

## Configuration

Copy configuration & migration to your project:

```
php artisan vendor:publish --provider="romanzipp\QueueMonitor\Providers\QueueMonitorProvider" --tag=config --tag=migrations
```

Migrate the Queue Monitoring table. The table name can be configured in the config file or via the published migration.

```
php artisan migrate
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

**Important!** You need to implement the `Illuminate\Contracts\Queue\ShouldQueue` interface to your job class. Otherwise, Laravel will not dispatch any events containing status information for monitoring the job.

## Web Interface

You can enable the web UI by setting the [`ui.enabled`](config/queue-monitor.php#23) to `true` configuration value.

**Publish frontend assets:**

```sh
php artisan vendor:publish --provider="romanzipp\QueueMonitor\Providers\QueueMonitorProvider" --tag=assets
```

See the [full configuration file](config/queue-monitor.php) for more information.

![Preview](preview.png#gh-light-mode-only)
![Preview](preview.dark.png#gh-dark-mode-only)

## Commands

### `artisan queue-monitor:stale`

This command mark old monitor entries as `stale`. You should run this command regularly in your console kernel.

**Arguments & Options:**
- `--before={date}` The start date before which all entries will be marked as stale
- `--beforeDays={days}` The relative amount of days before which entries will be marked as stale
- `--beforeInterval={interval}` An [interval date string](https://www.php.net/manual/en/dateinterval.createfromdatestring.php) before which entries will be marked as stale
- `--dry` Dry-run only

### `artisan queue-monitor:purge`

This command deletes old monitor models.

**Arguments & Options:**
- `--before={date}` The start date before which all entries will be deleted
- `--beforeDays={days}` The relative amount of days before which entries will be deleted
- `--beforeInterval={interval}` An [interval date string](https://www.php.net/manual/en/dateinterval.createfromdatestring.php) before which entries will be deleted
- `--queue={queue}` Only purge certain queues (comma separated values)
- `--only-succeeded` Only purge succeeded entries
- `--dry` Dry-run only

Either the **before** or **beforeDate** arguments are required.

## Advanced usage

### Progress

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

### Chunk progress

A common scenario for a job is iterating through large collections.

This example job loops through a large amount of users and updates its progress value with each chunk iteration.

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Database\Eloquent\Collection;
use romanzipp\QueueMonitor\Traits\IsMonitored;

class ChunkJob implements ShouldQueue
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

### Progress cooldown

To avoid flooding the database with rapidly repeating update queries, you can set override the `progressCooldown` method and specify a length in seconds to wait before each progress update is written to the database. Notice that cooldown will always be ignore for the values 0, 25, 50, 75 and 100.

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use romanzipp\QueueMonitor\Traits\IsMonitored;

class LazyJob implements ShouldQueue
{
    use IsMonitored;

    public function progressCooldown(): int
    {
        return 10; // Wait 10 seconds between each progress update
    }
}
``` 

### Custom data

This package also allows setting custom data in array syntax on the monitoring model.

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use romanzipp\QueueMonitor\Traits\IsMonitored;

class CustomDataJob implements ShouldQueue
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

In order to show custom data on UI you need to add this line under `config/queue-monitor.php`
```php
'ui' => [
    ...

    'show_custom_data' => true,

    ...
]
```

### Only keep failed jobs

You can override the `keepMonitorOnSuccess()` method to only store failed monitor entries of an executed job. This can be used if you only want to keep  failed monitors for jobs that are frequently executed but worth to monitor. Alternatively you can use Laravel's built in `failed_jobs` table.

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use romanzipp\QueueMonitor\Traits\IsMonitored;

class FrequentSucceedingJob implements ShouldQueue
{
    use IsMonitored;

    public static function keepMonitorOnSuccess(): bool
    {
        return false;
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
$job->getRemainingInterval(); // Carbon\CarbonInterval

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

## Tests

The easiest way to execute tests locally is via [**Lando**](https://lando.dev/). The [Lando config file](.lando.yml) automatically spins up app & database containers.

```shell
lando start

lando phpunit-mysql
lando phpunit-postgres
```

## Upgrading

- [**Upgrade from 2.0 to 3.0**](https://github.com/romanzipp/Laravel-Queue-Monitor/releases/tag/3.0.0)
- [Upgrade from 1.0 to 2.0](https://github.com/romanzipp/Laravel-Queue-Monitor/releases/tag/2.0.0)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

----

This package was inspired by gilbitron's [laravel-queue-monitor](https://github.com/gilbitron/laravel-queue-monitor) which is not maintained anymore.
