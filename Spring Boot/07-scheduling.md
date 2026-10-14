# 07 — Scheduling & Background Jobs in Spring Boot

> A complete, interview-ready deep dive into `@Scheduled`, Quartz, ShedLock, Spring Batch, and distributed scheduling patterns.

---

## Table of Contents

1. [@EnableScheduling & @Scheduled — The Basics](#1-enablescheduling--scheduled--the-basics)
2. [fixedRate vs fixedDelay vs cron — Critical Differences](#2-fixedrate-vs-fixeddelay-vs-cron--critical-differences)
3. [Cron Expression Mastery (Spring's 6-Field Syntax)](#3-cron-expression-mastery-springs-6-field-syntax)
4. [@Scheduled Threading Model — The #1 Interview Trap](#4-scheduled-threading-model--the-1-interview-trap)
5. [Dynamic Scheduling with SchedulingConfigurer](#5-dynamic-scheduling-with-schedulingconfigurer)
6. [Exception Handling in Scheduled Tasks](#6-exception-handling-in-scheduled-tasks)
7. [Quartz Scheduler — Enterprise-Grade Scheduling](#7-quartz-scheduler--enterprise-grade-scheduling)
8. [Distributed Scheduling Problems & ShedLock](#8-distributed-scheduling-problems--shedlock)
9. [Spring Batch — Batch Processing at Scale](#9-spring-batch--batch-processing-at-scale)
10. [Production Considerations](#10-production-considerations)
11. [⚠️ Interview Traps & Gotchas](#11-️-interview-traps--gotchas)
12. [🔍 Deep Dive — Internal Mechanics](#12--deep-dive--internal-mechanics)
13. [📝 Summary & Key Takeaways](#13--summary--key-takeaways)
14. [❓ Practice Questions](#14--practice-questions)

---

## 1. @EnableScheduling & @Scheduled — The Basics

### 1.1 Enabling the Scheduler

Spring's scheduling support is **not enabled by default**. You must opt in by adding `@EnableScheduling` to a configuration class.

```java
// Option 1: On your main application class (most common)
@SpringBootApplication
@EnableScheduling
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}

// Option 2: On a dedicated configuration class (better separation of concerns)
@Configuration
@EnableScheduling
public class SchedulingConfig {
    // Optionally configure TaskScheduler here (see Section 4)
}
```

> [!IMPORTANT]
> `@EnableScheduling` triggers a post-processor (`ScheduledAnnotationBeanPostProcessor`) that scans all Spring-managed beans for `@Scheduled` methods and registers them with the `TaskScheduler`. Without this annotation, **zero** scheduled tasks will run — no error is thrown.

### 1.2 Your First @Scheduled Method

```java
@Component
public class ReportGenerator {

    private static final Logger log = LoggerFactory.getLogger(ReportGenerator.class);

    @Scheduled(fixedRate = 5000) // every 5 seconds
    public void generateDailyReport() {
        log.info("Generating report at {}", LocalDateTime.now());
        // business logic here
    }
}
```

**Rules for `@Scheduled` methods:**
- Must have **no parameters** (unless using `SchedulingConfigurer`)
- Must have a **void** return type (return values are silently ignored)
- Must be on a **Spring-managed bean** (`@Component`, `@Service`, etc.)
- Must be **public** (can work on package-private with proxies, but avoid it)

> [!WARNING]
> Putting `@Scheduled` on a method in a class that is **not a Spring bean** (e.g., instantiated with `new`) will result in the task silently never running. This is one of the most common mistakes in interviews and production code.

---

## 2. fixedRate vs fixedDelay vs cron — Critical Differences

This is the most tested concept in Spring scheduling interviews. The differences are subtle but critical in production.

### 2.1 fixedRate

`fixedRate` schedules the task at a **fixed rate** measured from the **start time** of the previous execution.

```java
@Scheduled(fixedRate = 5000) // ms
public void fixedRateTask() {
    log.info("fixedRate task started at {}", System.currentTimeMillis());
    // Simulate work
    Thread.sleep(2000);
    log.info("fixedRate task finished");
}
```

**Timeline (single-threaded):**
```
T=0s  → Task starts
T=2s  → Task ends
T=5s  → Next task starts  (5s from start of previous)
T=7s  → Task ends
T=10s → Next task starts
```

> [!WARNING]
> In a single-threaded scheduler (the default), if the task takes **longer** than `fixedRate`, the next execution is **not** triggered concurrently — it is queued and fires immediately after the current one finishes. This is a critical production gotcha.

```
fixedRate=3s, task takes 7s:

T=0s  → Task1 starts
T=7s  → Task1 ends  (was supposed to fire at T=3s, T=6s — all missed)
T=7s  → Task2 starts immediately (missed executions NOT replayed)
T=14s → Task2 ends
T=14s → Task3 starts immediately
```

### 2.2 fixedDelay

`fixedDelay` schedules the task with a **fixed delay** measured from the **completion time** of the previous execution.

```java
@Scheduled(fixedDelay = 5000) // ms
public void fixedDelayTask() {
    log.info("fixedDelay task started at {}", System.currentTimeMillis());
    Thread.sleep(2000);
    log.info("fixedDelay task finished");
}
```

**Timeline:**
```
T=0s  → Task starts
T=2s  → Task ends
T=7s  → Next task starts  (2s end + 5s delay)
T=9s  → Task ends
T=14s → Next task starts  (9s end + 5s delay)
```

`fixedDelay` guarantees a **minimum gap** between executions. This is what you want when the task duration is variable and you need breathing room between runs.

### 2.3 cron

`cron` uses a cron expression to schedule tasks at specific calendar times.

```java
@Scheduled(cron = "0 30 9 * * MON-FRI") // 9:30 AM every weekday
public void cronTask() {
    log.info("Weekday morning task running");
}
```

### 2.4 Comparison Table

| Feature | `fixedRate` | `fixedDelay` | `cron` |
|---|---|---|---|
| Measured from | Task **start** time | Task **end** time | Calendar time |
| Missed executions | Fires immediately after current | Waits full delay after current | Skips missed windows |
| Variable task duration | ⚠️ Can overlap (with async) | ✅ Always waits | ✅ Fixed schedule |
| Good for | Heartbeats, polling | Sequential cleanup jobs | Business logic at specific times |
| Calendar awareness | ❌ No | ❌ No | ✅ Yes (weekdays, months, etc.) |
| Readable | ❌ Milliseconds | ❌ Milliseconds | ✅ Human-readable |

### 2.5 initialDelay

`initialDelay` specifies how long to wait before the **first** execution. Useful when the application needs time to warm up before running tasks.

```java
@Scheduled(fixedRate = 60_000, initialDelay = 30_000)
public void taskWithWarmup() {
    // Waits 30 seconds after startup, then runs every 60 seconds
    syncWithExternalService();
}
```

```java
// Reading delay from properties
@Scheduled(
    fixedRateString = "${app.scheduler.rate:60000}",
    initialDelayString = "${app.scheduler.initial-delay:30000}"
)
public void configurableTask() {
    processQueue();
}
```

### 2.6 Reading Values from Properties

Always externalize scheduling parameters:

```java
// application.yml
// app:
//   cron:
//     report: "0 0 8 * * MON-FRI"
//     cleanup: "0 0 2 * * *"

@Component
public class ScheduledJobs {

    @Scheduled(cron = "${app.cron.report}")
    public void morningReport() {
        generateAndEmailReport();
    }

    @Scheduled(cron = "${app.cron.cleanup}")
    public void nightlyCleanup() {
        archiveOldRecords();
    }

    // fixedRate from properties
    @Scheduled(fixedRateString = "${app.scheduler.heartbeat-rate:30000}")
    public void heartbeat() {
        checkServiceHealth();
    }
}
```

```yaml
# application.yml
app:
  cron:
    report: "0 0 8 * * MON-FRI"
    cleanup: "0 0 2 * * *"
  scheduler:
    heartbeat-rate: 30000
```

### 2.7 Zone Configuration

By default, cron expressions use the JVM's default timezone. Override with `zone`:

```java
@Scheduled(cron = "0 0 9 * * *", zone = "America/New_York")
public void newYorkTask() {
    // Runs at 9 AM Eastern regardless of server timezone
}

@Scheduled(cron = "0 0 18 * * *", zone = "Europe/London")
public void londonTask() {
    // Runs at 6 PM London time
}
```

> [!TIP]
> In containerized environments (Docker, Kubernetes), the container timezone might differ from the expected business timezone. Always set `zone` explicitly for business-critical jobs rather than relying on the JVM default.

---

## 3. Cron Expression Mastery (Spring's 6-Field Syntax)

### 3.1 Spring's 6-Field Cron vs Unix 5-Field Cron

This is a **critical interview distinction**:

| | Spring `@Scheduled` cron | Unix/Linux cron | Quartz cron |
|---|---|---|---|
| Fields | **6** | **5** | **7** (optional year) |
| Seconds field | ✅ Yes (first) | ❌ No | ✅ Yes (first) |
| Year field | ❌ No | ❌ No | ✅ Optional (last) |

```
Spring @Scheduled cron format:
┌─────────────── second       (0-59)
│ ┌───────────── minute       (0-59)
│ │ ┌─────────── hour         (0-23)
│ │ │ ┌───────── day-of-month (1-31)
│ │ │ │ ┌─────── month        (1-12 or JAN-DEC)
│ │ │ │ │ ┌───── day-of-week  (0-7 or MON-SUN, 0 and 7 are Sunday)
│ │ │ │ │ │
* * * * * *
```

```
Unix cron format (5 fields, NO seconds):
┌───────────── minute       (0-59)
│ ┌─────────── hour         (0-23)
│ │ ┌───────── day-of-month (1-31)
│ │ │ ┌─────── month        (1-12)
│ │ │ │ ┌───── day-of-week  (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *
```

> [!WARNING]
> If you copy a 5-field cron expression from a Unix system into `@Scheduled`, Spring will **throw an IllegalStateException** because it expects 6 fields. This is a common production bug.

### 3.2 Field-by-Field Breakdown

#### Special Characters

| Character | Meaning | Example |
|---|---|---|
| `*` | Every value | `* * * * * *` — every second |
| `,` | List of values | `0,15,30,45 * * * * *` — every 15 seconds |
| `-` | Range of values | `0 9-17 * * * *` — every minute, 9am to 5pm |
| `/` | Increment/step | `0 */15 * * * *` — every 15 minutes |
| `?` | No specific value (day-of-month or day-of-week) | `0 0 12 ? * MON` |
| `L` | Last (Quartz only) | `0 0 12 L * ?` — last day of month |
| `W` | Weekday nearest (Quartz only) | `0 0 9 15W * ?` — nearest weekday to 15th |
| `#` | Nth occurrence (Quartz only) | `0 0 10 ? * 2#1` — first Monday |

### 3.3 Common Cron Expressions

```java
// Every second
@Scheduled(cron = "* * * * * *")

// Every 5 seconds
@Scheduled(cron = "*/5 * * * * *")

// Every minute (at second 0)
@Scheduled(cron = "0 * * * * *")

// Every 15 minutes
@Scheduled(cron = "0 */15 * * * *")

// Every hour at :30
@Scheduled(cron = "0 30 * * * *")

// At midnight every day
@Scheduled(cron = "0 0 0 * * *")

// At 2:30 AM every day
@Scheduled(cron = "0 30 2 * * *")

// At 9 AM Monday through Friday
@Scheduled(cron = "0 0 9 * * MON-FRI")

// At noon on the 1st of every month
@Scheduled(cron = "0 0 12 1 * *")

// At 6 PM every Sunday
@Scheduled(cron = "0 0 18 * * SUN")

// At 8 AM every weekday (MON-FRI)
@Scheduled(cron = "0 0 8 * * 1-5")

// Every 30 minutes between 9 AM and 5 PM on weekdays
@Scheduled(cron = "0 */30 9-17 * * MON-FRI")

// First day of every month at midnight
@Scheduled(cron = "0 0 0 1 * *")

// At 11:59 PM on December 31
@Scheduled(cron = "0 59 23 31 12 *")

// Every quarter (Jan 1, Apr 1, Jul 1, Oct 1) at midnight
@Scheduled(cron = "0 0 0 1 1,4,7,10 *")
```

### 3.4 Cron Gotchas

```java
// ❌ WRONG: This is a Unix 5-field cron — Spring will REJECT this
@Scheduled(cron = "*/5 * * * *") // Missing seconds field

// ✅ CORRECT: Spring 6-field cron
@Scheduled(cron = "0 */5 * * * *") // Every 5 minutes

// ❌ WRONG: Day-of-week conflict — using both day-of-month AND day-of-week
// Spring supports '?' for "don't care" just like Quartz
// If both are specified, behavior might differ from expectation
@Scheduled(cron = "0 0 9 15 * MON") // ambiguous

// ✅ BETTER: Use ? for the field you don't care about
// (Spring 6 supports ? in both fields)
@Scheduled(cron = "0 0 9 15 * ?")  // 9 AM on the 15th
@Scheduled(cron = "0 0 9 ? * MON") // 9 AM on Mondays
```

> [!TIP]
> Use **[crontab.guru](https://crontab.guru)** (5-field) or **[cronmaker.com](http://www.cronmaker.com)** (6-field/Quartz) to validate your expressions. Remember to add the seconds field when moving from Unix to Spring cron format.

---

## 4. @Scheduled Threading Model — The #1 Interview Trap

### 4.1 Default: Single-Threaded Scheduler

By default, Spring creates a **single-threaded** `ThreadPoolTaskScheduler` with **1 thread** for ALL scheduled tasks in the application.

```
Application with 5 @Scheduled methods:
  ┌──────────────────────────────────────────┐
  │  Single Scheduler Thread                  │
  │                                           │
  │  taskA() ──────►  taskB() ──► taskC() ──► │
  │                                           │
  │  (All tasks run sequentially!)            │
  └──────────────────────────────────────────┘
```

**What this means:**
- If `taskA()` takes 10 seconds but is scheduled every 2 seconds, `taskB()` won't run until `taskA()` completes
- All `fixedRate` tasks will behave like `fixedDelay` if they run longer than their rate
- One slow/hanging task **blocks all other scheduled tasks**

### 4.2 Demonstrating the Problem

```java
@Component
public class BlockingSchedulerDemo {

    private static final Logger log = LoggerFactory.getLogger(BlockingSchedulerDemo.class);

    // Intended: runs every 1 second
    @Scheduled(fixedRate = 1000)
    public void fastTask() {
        log.info("[FAST] Running at {}", LocalDateTime.now());
    }

    // This takes 5 seconds! Blocks the single thread for 5s
    @Scheduled(fixedRate = 3000)
    public void slowTask() {
        log.info("[SLOW] Started at {}", LocalDateTime.now());
        try {
            Thread.sleep(5000); // Simulates slow I/O
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        log.info("[SLOW] Finished at {}", LocalDateTime.now());
    }
}
```

**Actual output (surprising!):**
```
[FAST] Running at 10:00:00
[SLOW] Started at 10:00:01
[SLOW] Finished at 10:00:06  ← blocked for 5 seconds
[FAST] Running at 10:00:06  ← fastTask was delayed by 5 seconds!
[SLOW] Started at 10:00:06
```

### 4.3 Configuring a Multi-Threaded TaskScheduler

**Fix 1: Configure a larger thread pool via properties (Spring Boot 2.4+)**

```yaml
# application.yml
spring:
  task:
    scheduling:
      pool:
        size: 10
      thread-name-prefix: scheduler-
      shutdown:
        await-termination: true
        await-termination-period: 60s
```

**Fix 2: Programmatic TaskScheduler bean**

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("MyApp-Scheduler-");
        scheduler.setAwaitTerminationSeconds(60);
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setErrorHandler(new CustomSchedulingErrorHandler()); // see Section 6
        scheduler.initialize();
        return scheduler;
    }
}
```

> [!IMPORTANT]
> Even with a multi-threaded pool, by default **the same `@Scheduled` method** will not run concurrently with itself. To allow concurrent self-execution, you must add `@Async` (see below).

### 4.4 @Async + @Scheduled Combination

Adding `@Async` to a `@Scheduled` method allows it to run in a **separate thread pool**, freeing the scheduler thread immediately.

```java
@Configuration
@EnableScheduling
@EnableAsync
public class AppConfig {

    @Bean("schedulerPool")
    public Executor schedulerAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-Scheduler-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Component
public class AsyncScheduledTasks {

    private static final Logger log = LoggerFactory.getLogger(AsyncScheduledTasks.class);

    // The scheduler fires every 5s, but execution happens on a separate thread pool
    // This allows concurrent executions of the same task if it runs longer than 5s
    @Async("schedulerPool")
    @Scheduled(fixedRate = 5000)
    public void concurrentTask() {
        log.info("Task running on thread: {}", Thread.currentThread().getName());
        processLargeDataset();
    }
}
```

> [!WARNING]
> `@Async` + `@Scheduled` means the task CAN run concurrently with itself. If your task takes longer than `fixedRate`, multiple instances of it will be running simultaneously. Ensure the task is thread-safe and idempotent, or use `fixedDelay` instead.

**Thread execution model comparison:**

```
Without @Async (default, single-threaded):
Scheduler Thread: [Task1 ──────────────][Task1 ──────────────][Task1]

With @Async + fixedRate=5s, task takes 8s:
Scheduler Thread: [dispatch][dispatch][dispatch]...
Worker Thread 1:  [Task1 ════════════════════]
Worker Thread 2:          [Task1 ════════════════════]  ← concurrent!
Worker Thread 3:                  [Task1 ════════════════════]
```

---

## 5. Dynamic Scheduling with SchedulingConfigurer

Sometimes you need to change the schedule at runtime (e.g., read from DB, toggle features). Use `SchedulingConfigurer` for this.

### 5.1 Basic SchedulingConfigurer

```java
@Configuration
@EnableScheduling
public class DynamicSchedulingConfig implements SchedulingConfigurer {

    @Autowired
    private ScheduleRepository scheduleRepository; // DB-backed schedule config

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        registrar.setScheduler(dynamicTaskScheduler());

        // Add a task with a Trigger that re-evaluates the cron on each run
        registrar.addTriggerTask(
            new Runnable() {
                @Override
                public void run() {
                    performScheduledWork();
                }
            },
            new Trigger() {
                @Override
                public Instant nextExecution(TriggerContext context) {
                    // Re-read from DB on every trigger evaluation
                    String cronExpression = scheduleRepository.getCronExpression("main-job");
                    CronTrigger trigger = new CronTrigger(cronExpression);
                    return trigger.nextExecution(context);
                }
            }
        );
    }

    @Bean(destroyMethod = "shutdown")
    public Executor dynamicTaskScheduler() {
        return Executors.newScheduledThreadPool(5);
    }

    private void performScheduledWork() {
        // actual business logic
    }
}
```

### 5.2 Full Dynamic Scheduler with Runtime Updates

```java
@Service
public class DynamicSchedulerService {

    private final ScheduledTaskRegistrar registrar = new ScheduledTaskRegistrar();
    private final Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();
    private final ThreadPoolTaskScheduler taskScheduler;

    public DynamicSchedulerService() {
        this.taskScheduler = new ThreadPoolTaskScheduler();
        this.taskScheduler.setPoolSize(10);
        this.taskScheduler.initialize();
    }

    public void scheduleTask(String taskId, Runnable task, String cronExpression) {
        // Cancel existing task if present
        cancelTask(taskId);

        CronTrigger trigger = new CronTrigger(cronExpression);
        ScheduledFuture<?> future = taskScheduler.schedule(task, trigger);
        scheduledTasks.put(taskId, future);
        log.info("Scheduled task '{}' with cron '{}'", taskId, cronExpression);
    }

    public void cancelTask(String taskId) {
        ScheduledFuture<?> future = scheduledTasks.remove(taskId);
        if (future != null) {
            future.cancel(false); // false = don't interrupt if running
            log.info("Cancelled task '{}'", taskId);
        }
    }

    public void rescheduleTask(String taskId, Runnable task, String newCron) {
        scheduleTask(taskId, task, newCron);
    }
}

// REST controller to manage dynamic schedules
@RestController
@RequestMapping("/admin/schedules")
public class ScheduleController {

    @Autowired
    private DynamicSchedulerService schedulerService;

    @PostMapping("/{taskId}")
    public ResponseEntity<String> updateSchedule(
            @PathVariable String taskId,
            @RequestParam String cron) {

        schedulerService.scheduleTask(taskId, () -> runNamedTask(taskId), cron);
        return ResponseEntity.ok("Task '" + taskId + "' scheduled with: " + cron);
    }

    @DeleteMapping("/{taskId}")
    public ResponseEntity<String> cancelSchedule(@PathVariable String taskId) {
        schedulerService.cancelTask(taskId);
        return ResponseEntity.ok("Task '" + taskId + "' cancelled");
    }

    private void runNamedTask(String taskId) {
        log.info("Running dynamic task: {}", taskId);
    }
}
```

---

## 6. Exception Handling in Scheduled Tasks

### 6.1 Default Behavior: Task Dies Silently on Exception!

This is **the most dangerous default behavior** in Spring scheduling:

```java
@Scheduled(fixedRate = 5000)
public void dangerousTask() {
    log.info("Task running...");
    // If this throws any RuntimeException:
    riskyOperation(); // throws NullPointerException

    // The task will NEVER run again!
    // Spring logs the error but does not restart the task.
    // No alert is sent. The task dies silently.
}
```

**What actually happens:**
1. Task runs at T=0, throws an exception
2. Spring's `ScheduledAnnotationBeanPostProcessor` catches it and logs a warning
3. The task is marked as failed in the scheduler
4. **The task is never re-scheduled** — it's dead
5. No alert, no restart, just silence

### 6.2 Fixing with try-catch (Minimum Viable Solution)

```java
@Scheduled(fixedRate = 5000)
public void safeTask() {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Scheduled task failed: {}", e.getMessage(), e);
        // Optionally: send alert, increment counter, etc.
        // Task will still run next time because exception was swallowed
    }
}
```

### 6.3 Custom ErrorHandler (Recommended for Production)

```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {

    @Autowired
    private AlertService alertService;

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("Scheduler-");
        scheduler.setErrorHandler(new ScheduledTaskErrorHandler(alertService));
        scheduler.initialize();
        registrar.setTaskScheduler(scheduler);
    }
}

public class ScheduledTaskErrorHandler implements ErrorHandler {

    private static final Logger log = LoggerFactory.getLogger(ScheduledTaskErrorHandler.class);

    private final AlertService alertService;

    public ScheduledTaskErrorHandler(AlertService alertService) {
        this.alertService = alertService;
    }

    @Override
    public void handleError(Throwable t) {
        log.error("Scheduled task threw an exception: {}", t.getMessage(), t);
        // Task WILL be rescheduled (error is handled, not propagated)
        alertService.sendAlert(
            "Scheduled Task Failure",
            "A scheduled task failed: " + t.getMessage()
        );
        // Optionally: increment metrics counter
        Metrics.counter("scheduled.task.errors").increment();
    }
}
```

### 6.4 Production-Grade Scheduled Task with Full Error Handling

```java
@Component
public class RobustScheduledTask {

    private static final Logger log = LoggerFactory.getLogger(RobustScheduledTask.class);
    private final MeterRegistry meterRegistry;
    private final AlertService alertService;
    private final AtomicInteger consecutiveFailures = new AtomicInteger(0);
    private static final int MAX_FAILURES_BEFORE_ALERT = 3;

    public RobustScheduledTask(MeterRegistry meterRegistry, AlertService alertService) {
        this.meterRegistry = meterRegistry;
        this.alertService = alertService;
    }

    @Scheduled(cron = "0 0 2 * * *") // 2 AM daily
    public void nightlyDataSync() {
        Timer.Sample sample = Timer.start(meterRegistry);
        log.info("Starting nightly data sync at {}", LocalDateTime.now());

        try {
            performDataSync();
            consecutiveFailures.set(0); // reset on success
            log.info("Nightly data sync completed successfully");
            meterRegistry.counter("scheduler.nightly.sync.success").increment();
        } catch (DataAccessException e) {
            handleFailure("Database error during nightly sync", e);
        } catch (ExternalServiceException e) {
            handleFailure("External service unavailable during nightly sync", e);
        } catch (Exception e) {
            handleFailure("Unexpected error during nightly sync", e);
        } finally {
            sample.stop(meterRegistry.timer("scheduler.nightly.sync.duration"));
        }
    }

    private void handleFailure(String message, Exception e) {
        log.error("{}: {}", message, e.getMessage(), e);
        meterRegistry.counter("scheduler.nightly.sync.failure").increment();

        int failures = consecutiveFailures.incrementAndGet();
        if (failures >= MAX_FAILURES_BEFORE_ALERT) {
            alertService.sendCriticalAlert(message + " (failed " + failures + " times)", e);
        }
    }

    private void performDataSync() {
        // Actual business logic
    }
}
```

---

## 7. Quartz Scheduler — Enterprise-Grade Scheduling

### 7.1 @Scheduled vs Quartz — When to Use Each

| Feature | `@Scheduled` | Quartz |
|---|---|---|
| Setup complexity | Very simple | Moderate |
| Persistence | ❌ In-memory only | ✅ DB-backed (JDBC JobStore) |
| Clustering | ❌ Not supported | ✅ Built-in clustering |
| Job history | ❌ None | ✅ Via JobStore |
| Dynamic scheduling | Limited | ✅ Full API |
| Misfire handling | ❌ None | ✅ Configurable |
| Monitoring | Limited | ✅ JMX, REST API |
| Spring Boot auto-config | ✅ Yes | ✅ Yes (spring-boot-starter-quartz) |
| Use case | Simple, single-instance | Enterprise, multi-instance |

### 7.2 Quartz Core Concepts

```
Quartz Architecture:
┌────────────────────────────────────────────────────┐
│                   Scheduler                         │
│  ┌──────────────┐    ┌───────────────────────────┐  │
│  │  JobDetail   │    │         Trigger            │  │
│  │ ┌──────────┐ │    │  ┌────────────────────┐   │  │
│  │ │   Job    │ │◄───┤  │ SimpleTrigger /    │   │  │
│  │ │(business │ │    │  │ CronTrigger        │   │  │
│  │ │ logic)   │ │    │  └────────────────────┘   │  │
│  │ └──────────┘ │    └───────────────────────────┘  │
│  └──────────────┘                                   │
│                    ┌──────────┐                     │
│                    │JobStore  │                     │
│                    │(RAM/JDBC)│                     │
│                    └──────────┘                     │
└────────────────────────────────────────────────────┘
```

**Key Quartz concepts:**
- **`Job`**: Interface with a single `execute()` method — the actual work
- **`JobDetail`**: Metadata about a `Job` (identity, durability, job data map)
- **`Trigger`**: Defines when a job runs (SimpleTrigger for intervals, CronTrigger for cron)
- **`Scheduler`**: Orchestrates `JobDetail` + `Trigger` pairs
- **`JobDataMap`**: Key-value store passed to job at execution time
- **`JobExecutionContext`**: Runtime context passed to `execute()`

### 7.3 Adding Quartz to Spring Boot

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

### 7.4 Complete Example: Email Digest Job with Quartz

```java
// Step 1: Define the Job
@Component
public class EmailDigestJob implements Job {

    private static final Logger log = LoggerFactory.getLogger(EmailDigestJob.class);

    // Spring beans can be auto-wired when using SpringBeanJobFactory (auto-configured)
    @Autowired
    private EmailService emailService;

    @Autowired
    private UserRepository userRepository;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        // Get job parameters from JobDataMap
        JobDataMap dataMap = context.getMergedJobDataMap();
        String digestType = dataMap.getString("digestType");
        String recipientGroup = dataMap.getString("recipientGroup");

        log.info("Executing EmailDigestJob: type={}, group={}", digestType, recipientGroup);

        try {
            List<User> recipients = userRepository.findByGroup(recipientGroup);
            emailService.sendDigest(recipients, digestType);
            log.info("Email digest sent to {} recipients", recipients.size());
        } catch (Exception e) {
            // Re-throw as JobExecutionException to signal Quartz
            throw new JobExecutionException("Failed to send email digest", e);
        }
    }
}
```

```java
// Step 2: Configure JobDetail and Triggers
@Configuration
public class QuartzJobConfig {

    // JobDetail — describes the job, its identity, and data
    @Bean
    public JobDetail dailyDigestJobDetail() {
        return JobBuilder.newJob(EmailDigestJob.class)
            .withIdentity("dailyDigestJob", "emailJobs")
            .withDescription("Sends daily email digest to all users")
            .usingJobData("digestType", "DAILY")
            .usingJobData("recipientGroup", "ALL_USERS")
            .storeDurably() // Keep job even if no trigger is associated
            .build();
    }

    @Bean
    public JobDetail weeklyDigestJobDetail() {
        return JobBuilder.newJob(EmailDigestJob.class)
            .withIdentity("weeklyDigestJob", "emailJobs")
            .withDescription("Sends weekly summary to premium users")
            .usingJobData("digestType", "WEEKLY")
            .usingJobData("recipientGroup", "PREMIUM_USERS")
            .storeDurably()
            .build();
    }

    // CronTrigger — calendar-based scheduling
    @Bean
    public CronTrigger dailyDigestTrigger() {
        CronScheduleBuilder scheduleBuilder = CronScheduleBuilder
            .cronSchedule("0 0 8 * * ?")  // 8 AM every day (Quartz 6-field, no year)
            .withMisfireHandlingInstructionFireAndProceed(); // handle missed executions

        return TriggerBuilder.newTrigger()
            .forJob(dailyDigestJobDetail())
            .withIdentity("dailyDigestTrigger", "emailTriggers")
            .withDescription("Daily at 8 AM")
            .withSchedule(scheduleBuilder)
            .build();
    }

    // SimpleTrigger — interval-based scheduling
    @Bean
    public SimpleTrigger heartbeatTrigger() {
        SimpleScheduleBuilder scheduleBuilder = SimpleScheduleBuilder
            .simpleSchedule()
            .withIntervalInSeconds(30)
            .repeatForever()
            .withMisfireHandlingInstructionNextWithRemainingCount();

        return TriggerBuilder.newTrigger()
            .forJob("heartbeatJob", "systemJobs")
            .withIdentity("heartbeatTrigger", "systemTriggers")
            .withSchedule(scheduleBuilder)
            .build();
    }

    @Bean
    public CronTrigger weeklyDigestTrigger() {
        return TriggerBuilder.newTrigger()
            .forJob(weeklyDigestJobDetail())
            .withIdentity("weeklyDigestTrigger", "emailTriggers")
            .withSchedule(CronScheduleBuilder.cronSchedule("0 0 9 ? * MON")) // 9 AM Monday
            .build();
    }
}
```

### 7.5 Quartz Configuration in application.yml

```yaml
spring:
  quartz:
    job-store-type: jdbc         # Use database for persistence
    jdbc:
      initialize-schema: always  # Create Quartz tables automatically
    properties:
      org.quartz:
        scheduler:
          instanceName: MyAppScheduler
          instanceId: AUTO          # Auto-generate unique instance ID (for clustering)
        threadPool:
          class: org.quartz.simpl.SimpleThreadPool
          threadCount: 10
          threadPriority: 5
        jobStore:
          class: org.quartz.impl.jdbcjobstore.JobStoreTX
          driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
          useProperties: false
          dataSource: myDS
          tablePrefix: QRTZ_
          isClustered: true          # Enable clustering
          clusterCheckinInterval: 20000  # 20 seconds
        dataSource:
          myDS:
            driver: org.postgresql.Driver
            URL: jdbc:postgresql://localhost:5432/scheduler_db
            user: ${DB_USER}
            password: ${DB_PASSWORD}
            maxConnections: 5
```

### 7.6 JDBC JobStore — Persistent Jobs

When using `job-store-type: jdbc`, Quartz creates tables in your database:

```sql
-- Key Quartz tables (auto-created with initialize-schema: always)
QRTZ_JOB_DETAILS     -- Stores job definitions
QRTZ_TRIGGERS        -- Stores trigger definitions
QRTZ_SIMPLE_TRIGGERS -- SimpleTrigger data
QRTZ_CRON_TRIGGERS   -- CronTrigger data
QRTZ_FIRED_TRIGGERS  -- Currently executing triggers
QRTZ_SCHEDULER_STATE -- Cluster node heartbeats
QRTZ_LOCKS           -- Pessimistic locks for clustering
```

**Benefits of JDBC JobStore:**
- Jobs survive application restarts
- Triggers with missed fires are handled according to misfire policy
- Cluster-safe (only one node executes each job)
- Full audit trail of fired triggers

### 7.7 Clustering Quartz

```
Quartz Cluster:
                    ┌──────────────────────┐
                    │   PostgreSQL DB       │
                    │  (QRTZ_* tables)      │
                    │  - Job definitions    │
                    │  - Trigger state      │
                    │  - Cluster heartbeats │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼──────┐  ┌──────▼────────┐  ┌───▼───────────┐
     │   App Node 1  │  │  App Node 2   │  │  App Node 3   │
     │  (Active)     │  │  (Standby)    │  │  (Standby)    │
     │               │  │               │  │               │
     │  Running:     │  │               │  │               │
     │  emailDigest  │  │               │  │               │
     └───────────────┘  └───────────────┘  └───────────────┘

When Node 1 dies → Node 2 or 3 picks up pending jobs automatically
```

---

## 8. Distributed Scheduling Problems & ShedLock

### 8.1 The Multi-Instance Problem

When you deploy multiple instances of a Spring Boot app (e.g., 3 replicas in Kubernetes), **every instance runs every `@Scheduled` task**.

```
Problem: 3 app instances, all running nightly cleanup at 2 AM:

Instance 1: [Cleanup starts] → deletes 10,000 records
Instance 2: [Cleanup starts] → tries to delete same 10,000 records → errors
Instance 3: [Cleanup starts] → tries to delete same 10,000 records → errors

Result: Duplicate processing, constraint violations, data corruption
```

### 8.2 ShedLock — Simple Solution

ShedLock uses a **distributed lock** (backed by a database, Redis, Mongo, etc.) to ensure only one instance runs a scheduled task at a time.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.10.0</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>5.10.0</version>
</dependency>
```

**Database table for ShedLock:**

```sql
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL,
    lock_until TIMESTAMP    NOT NULL,
    locked_at  TIMESTAMP    NOT NULL,
    locked_by  VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

### 8.3 ShedLock Configuration

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "PT30M") // 30 minutes max lock
public class ShedLockConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime() // Use DB time to avoid clock skew between instances
                .build()
        );
    }
}
```

### 8.4 @SchedulerLock Annotation

```java
@Component
public class DistributedScheduledTasks {

    private static final Logger log = LoggerFactory.getLogger(DistributedScheduledTasks.class);

    @Scheduled(cron = "0 0 2 * * *") // 2 AM daily
    @SchedulerLock(
        name = "nightlyCleanupTask",        // Unique lock name
        lockAtLeastFor = "PT5M",            // Hold lock for at least 5 minutes
        lockAtMostFor = "PT1H"              // Release lock after 1 hour (failsafe)
    )
    public void nightlyCleanup() {
        log.info("Running nightly cleanup on instance: {}", getInstanceId());
        cleanupOldData();
        // Only ONE instance across the cluster will execute this
    }

    @Scheduled(fixedRate = 30_000) // Every 30 seconds
    @SchedulerLock(
        name = "dataPollingTask",
        lockAtLeastFor = "PT25S",   // Ensure minimum 25s gap between runs
        lockAtMostFor = "PT29S"     // Release before next scheduled run
    )
    public void pollExternalData() {
        fetchAndProcessExternalData();
    }

    @Scheduled(cron = "0 0 8 * * MON-FRI") // 8 AM weekdays
    @SchedulerLock(
        name = "weekdayReportTask",
        lockAtLeastFor = "PT10M",
        lockAtMostFor = "PT2H"
    )
    public void generateWeekdayReport() {
        generateAndDistributeReport();
    }
}
```

### 8.5 How ShedLock Works Internally

```
ShedLock execution flow:

Instance 1 at 2:00:00 AM:              Instance 2 at 2:00:00 AM:
  1. Try INSERT into shedlock table       1. Try INSERT into shedlock table
  2. INSERT succeeds → lock acquired!     2. INSERT fails (name already exists)
  3. Run nightlyCleanup()                 3. SKIP execution → return immediately
  4. UPDATE lock_until if lockAtLeastFor  4. Log: "Lock not obtained for task X"
  5. Task completes
  6. Lock expires after lockAtMostFor

shedlock table during execution:
name               | lock_until          | locked_at           | locked_by
nightlyCleanupTask | 2023-01-01 03:00:00 | 2023-01-01 02:00:00 | instance-1
```

> [!IMPORTANT]
> **`lockAtMostFor`** is a **safety net**: if the instance that holds the lock crashes before releasing it, the lock will automatically expire after this duration, allowing another instance to take over. Always set it to a value longer than the expected task duration.

> [!TIP]
> **`lockAtLeastFor`** prevents the issue where a very fast task completes and another instance immediately picks up the lock, causing rapid repeated executions. Set it to something close to your `fixedRate` interval.

### 8.6 ShedLock with Redis

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-redis-spring</artifactId>
    <version>5.10.0</version>
</dependency>
```

```java
@Bean
public LockProvider lockProvider(RedisConnectionFactory connectionFactory) {
    return new RedisLockProvider(connectionFactory, "my-app-locks");
}
```

---

## 9. Spring Batch — Batch Processing at Scale

### 9.1 When to Use Spring Batch vs @Scheduled

| Criteria | `@Scheduled` | Spring Batch |
|---|---|---|
| Processing millions of records | ❌ No built-in support | ✅ Chunk processing |
| Job restart on failure | ❌ Restarts from beginning | ✅ Restarts from last checkpoint |
| Job progress tracking | ❌ None | ✅ JobRepository |
| Parallel processing | Manual setup | ✅ Built-in partitioning |
| Input → Process → Output pattern | Manual | ✅ ItemReader/Processor/Writer |
| ETL operations | ❌ | ✅ Perfect fit |
| Audit trail | ❌ None | ✅ JobExecution history |
| Simple periodic tasks | ✅ Perfect | ❌ Overkill |

### 9.2 Spring Batch Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
```

### 9.3 Core Spring Batch Architecture

```
Spring Batch Architecture:
┌─────────────────────────────────────────────────┐
│                    Job                           │
│  ┌──────────────────────────────────────────┐   │
│  │              Step 1                       │   │
│  │  ┌──────────┐ ┌───────────┐ ┌──────────┐ │   │
│  │  │   Item   │ │   Item    │ │   Item   │ │   │
│  │  │  Reader  │►│ Processor │►│  Writer  │ │   │
│  │  └──────────┘ └───────────┘ └──────────┘ │   │
│  │         ▲ Chunk (e.g., 100 records) ▲    │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │              Step 2 (optional)            │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │   JobRepository     │
              │  (DB: job history,  │
              │   step progress,    │
              │   restart data)     │
              └─────────────────────┘
```

### 9.4 Complete Spring Batch Example: Process User Records

```java
// Domain model
public class UserRecord {
    private Long id;
    private String email;
    private String status;
    private LocalDate lastLogin;
    // getters, setters, constructors
}

// ItemReader: Read inactive users from DB
@Component
@StepScope
public class InactiveUserReader implements ItemReader<UserRecord> {

    private final JdbcTemplate jdbcTemplate;
    private final List<UserRecord> buffer = new ArrayList<>();
    private int currentIndex = 0;

    public InactiveUserReader(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @BeforeStep
    public void beforeStep(StepExecution stepExecution) {
        // Load all inactive users (>90 days no login) into buffer
        LocalDate cutoff = LocalDate.now().minusDays(90);
        List<UserRecord> users = jdbcTemplate.query(
            "SELECT id, email, status, last_login FROM users WHERE last_login < ? AND status = 'ACTIVE'",
            new BeanPropertyRowMapper<>(UserRecord.class),
            cutoff
        );
        buffer.addAll(users);
    }

    @Override
    public UserRecord read() {
        if (currentIndex < buffer.size()) {
            return buffer.get(currentIndex++);
        }
        return null; // null signals end of data
    }
}

// ItemProcessor: Transform/validate each record
@Component
@StepScope
public class InactiveUserProcessor implements ItemProcessor<UserRecord, UserRecord> {

    private static final Logger log = LoggerFactory.getLogger(InactiveUserProcessor.class);

    @Override
    public UserRecord process(UserRecord user) throws Exception {
        // Return null to SKIP the item (won't be passed to writer)
        if (user.getEmail() == null || user.getEmail().isBlank()) {
            log.warn("Skipping user {} with no email", user.getId());
            return null;
        }

        // Transform: mark as inactive
        user.setStatus("INACTIVE");
        return user;
    }
}

// ItemWriter: Write processed records back to DB
@Component
@StepScope
public class InactiveUserWriter implements ItemWriter<UserRecord> {

    private final JdbcTemplate jdbcTemplate;

    public InactiveUserWriter(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public void write(Chunk<? extends UserRecord> chunk) throws Exception {
        // Chunk is a batch of processed items (e.g., 100 records)
        List<Object[]> batchArgs = chunk.getItems().stream()
            .map(u -> new Object[]{u.getStatus(), u.getId()})
            .collect(Collectors.toList());

        jdbcTemplate.batchUpdate(
            "UPDATE users SET status = ? WHERE id = ?",
            batchArgs
        );
    }
}
```

```java
// Job Configuration
@Configuration
public class InactiveUserJobConfig {

    @Bean
    public Job inactiveUserCleanupJob(
            JobRepository jobRepository,
            Step inactiveUserStep) {

        return new JobBuilder("inactiveUserCleanupJob", jobRepository)
            .incrementer(new RunIdIncrementer()) // Allow repeated runs
            .start(inactiveUserStep)
            .build();
    }

    @Bean
    public Step inactiveUserStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            InactiveUserReader reader,
            InactiveUserProcessor processor,
            InactiveUserWriter writer) {

        return new StepBuilder("inactiveUserStep", jobRepository)
            .<UserRecord, UserRecord>chunk(100, transactionManager) // Process 100 at a time
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skip(DataAccessException.class) // Skip records with DB errors
            .skipLimit(10)                   // Skip at most 10 records total
            .retry(OptimisticLockingFailureException.class)
            .retryLimit(3)
            .listener(new StepExecutionListener() {
                @Override
                public ExitStatus afterStep(StepExecution stepExecution) {
                    log.info("Step finished: read={}, written={}, skipped={}",
                        stepExecution.getReadCount(),
                        stepExecution.getWriteCount(),
                        stepExecution.getSkipCount());
                    return stepExecution.getExitStatus();
                }
            })
            .build();
    }
}
```

### 9.5 Triggering Spring Batch Jobs with @Scheduled

```java
@Component
public class BatchJobScheduler {

    private final JobLauncher jobLauncher;
    private final Job inactiveUserCleanupJob;

    public BatchJobScheduler(JobLauncher jobLauncher, Job inactiveUserCleanupJob) {
        this.jobLauncher = jobLauncher;
        this.inactiveUserCleanupJob = inactiveUserCleanupJob;
    }

    @Scheduled(cron = "0 0 3 * * *") // 3 AM daily
    public void runInactiveUserCleanup() {
        try {
            JobParameters params = new JobParametersBuilder()
                .addLocalDateTime("startTime", LocalDateTime.now()) // Unique params for each run
                .addString("triggeredBy", "scheduler")
                .toJobParameters();

            JobExecution execution = jobLauncher.run(inactiveUserCleanupJob, params);
            log.info("Job '{}' completed with status: {}",
                inactiveUserCleanupJob.getName(),
                execution.getStatus());

        } catch (JobExecutionAlreadyRunningException e) {
            log.warn("Job is already running, skipping this trigger");
        } catch (Exception e) {
            log.error("Failed to launch batch job", e);
        }
    }
}
```

```yaml
# application.yml — Disable auto-run of batch jobs at startup
spring:
  batch:
    job:
      enabled: false  # Don't auto-run jobs on startup
    jdbc:
      initialize-schema: always  # Create batch tables
```

---

## 10. Production Considerations

### 10.1 Graceful Shutdown

Spring Boot's graceful shutdown waits for in-flight tasks to complete before stopping.

```yaml
# application.yml
server:
  shutdown: graceful   # Enable graceful shutdown

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # Wait up to 30s for tasks to finish

  task:
    scheduling:
      shutdown:
        await-termination: true
        await-termination-period: 60s  # Wait for scheduled tasks
    execution:
      shutdown:
        await-termination: true
        await-termination-period: 60s  # Wait for @Async tasks
```

```java
@Configuration
@EnableScheduling
public class SchedulerConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("Scheduler-");
        // Critical for graceful shutdown:
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(60);
        scheduler.initialize();
        return scheduler;
    }
}
```

### 10.2 Monitoring Scheduled Tasks

```java
@Component
public class MonitoredScheduledTask {

    private final MeterRegistry meterRegistry;
    private volatile Instant lastSuccessfulRun;
    private volatile Instant lastFailedRun;

    public MonitoredScheduledTask(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        // Register gauges for monitoring
        Gauge.builder("scheduler.task.seconds.since.last.success",
                this, task -> {
                    if (task.lastSuccessfulRun == null) return -1.0;
                    return Duration.between(task.lastSuccessfulRun, Instant.now()).toSeconds();
                })
            .description("Seconds since last successful execution")
            .register(meterRegistry);
    }

    @Scheduled(cron = "0 */5 * * * *") // Every 5 minutes
    public void monitoredTask() {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            doWork();
            lastSuccessfulRun = Instant.now();
            meterRegistry.counter("scheduler.task.success").increment();
        } catch (Exception e) {
            lastFailedRun = Instant.now();
            meterRegistry.counter("scheduler.task.failure").increment();
            throw e; // Let ErrorHandler deal with it
        } finally {
            sample.stop(meterRegistry.timer("scheduler.task.duration"));
        }
    }

    private void doWork() {
        // Business logic
    }
}
```

### 10.3 Custom Health Indicator for Scheduled Jobs

```java
@Component
public class ScheduledJobHealthIndicator implements HealthIndicator {

    private volatile Instant lastRun;
    private volatile String lastStatus = "UNKNOWN";
    private final Duration maxAcceptableGap;

    public ScheduledJobHealthIndicator(
            @Value("${scheduler.health.max-gap:PT10M}") Duration maxAcceptableGap) {
        this.maxAcceptableGap = maxAcceptableGap;
    }

    public void recordSuccess() {
        this.lastRun = Instant.now();
        this.lastStatus = "SUCCESS";
    }

    public void recordFailure(Exception e) {
        this.lastRun = Instant.now();
        this.lastStatus = "FAILED: " + e.getMessage();
    }

    @Override
    public Health health() {
        if (lastRun == null) {
            return Health.unknown()
                .withDetail("status", "Never run")
                .build();
        }

        Duration gap = Duration.between(lastRun, Instant.now());

        if (gap.compareTo(maxAcceptableGap) > 0) {
            return Health.down()
                .withDetail("lastRun", lastRun)
                .withDetail("gap", gap.toMinutes() + " minutes")
                .withDetail("status", lastStatus)
                .withDetail("issue", "Task hasn't run in " + gap.toMinutes() + " minutes")
                .build();
        }

        return Health.up()
            .withDetail("lastRun", lastRun)
            .withDetail("lastStatus", lastStatus)
            .withDetail("secondsAgo", gap.toSeconds())
            .build();
    }
}
```

### 10.4 Idempotency of Scheduled Tasks

All scheduled tasks should be **idempotent** — running them multiple times should produce the same result.

```java
@Scheduled(cron = "0 0 8 * * MON-FRI")
public void generateMorningReport() {
    // ❌ BAD: Not idempotent — generates duplicate reports if run twice
    Report report = generateReport();
    emailService.send(report);

    // ✅ GOOD: Idempotent — check if already run for today
    String reportDate = LocalDate.now().toString();
    if (reportRepository.existsByDate(reportDate)) {
        log.info("Report for {} already generated, skipping", reportDate);
        return;
    }
    Report report2 = generateReport();
    report2.setDate(reportDate);
    reportRepository.save(report2);
    emailService.send(report2);
}
```

> [!TIP]
> Design your scheduled tasks assuming they might run multiple times due to restarts, clustering issues, or manual triggers. Use a **unique constraint** or **execution log** to detect and skip duplicate runs.

---

## 11. ⚠️ Interview Traps & Gotchas

### Trap 1: @Scheduled Default Thread Count

**Question:** "How many threads does Spring use for `@Scheduled` by default?"

**Wrong answer:** "One per task" or "Multiple threads"

**Correct answer:** **ONE** thread for ALL `@Scheduled` tasks in the entire application. The default `ThreadPoolTaskScheduler` has a pool size of 1. This means all scheduled tasks run sequentially on a single thread, and a slow task will block all other tasks.

```java
// Both tasks share the SAME single thread by default:
@Scheduled(fixedRate = 1000)
public void task1() { /* ... */ }

@Scheduled(fixedRate = 2000)
public void task2() { /* ... */ }
```

**Fix:** Configure `spring.task.scheduling.pool.size` to a value greater than 1.

---

### Trap 2: fixedRate Task Longer Than the Rate

**Question:** "What happens if a `fixedRate = 1000` task takes 5 seconds to complete?"

**Wrong answer:** "It will run concurrently with itself"

**Correct answer (single-threaded):** The next execution is **queued** but waits for the current one to finish. With a single thread, no concurrent execution occurs. The task effectively behaves like `fixedDelay` when it runs long. **No executions are missed** — they are queued.

**Correct answer (with `@Async`):** With `@Async`, each invocation runs on a separate thread pool thread. Multiple instances CAN run concurrently.

---

### Trap 3: @Scheduled on a Non-Spring-Managed Bean

```java
// ❌ This WILL NOT work — MyTask is not a Spring bean
public class MyTask {
    @Scheduled(fixedRate = 1000)
    public void run() { /* Never called */ }
}

// Somewhere in code:
MyTask task = new MyTask(); // Instantiated with new, not by Spring

// ✅ Must be a Spring bean:
@Component
public class MyTask {
    @Scheduled(fixedRate = 1000)
    public void run() { /* Called every second */ }
}
```

---

### Trap 4: @Scheduled with @Async — Does It Still Block?

**Question:** "If I add `@Async` to a `@Scheduled` method, does the scheduler thread still block?"

**Correct answer:** No. With `@Async`, the scheduler thread submits the task to the async executor's thread pool and returns immediately. The scheduler thread is freed. However, this means the same task CAN run concurrently with itself if it takes longer than the scheduled interval.

```java
// Without @Async: scheduler thread blocks for 10 seconds
// With @Async: scheduler thread returns immediately, work done on async pool
@Async
@Scheduled(fixedRate = 5000)
public void asyncTask() {
    Thread.sleep(10000); // 10-second task
}
```

---

### Trap 5: @Scheduled in Multi-Instance Deployment

**Question:** "You deploy 3 instances of your Spring Boot app. A `@Scheduled` task runs nightly. How many times will it run?"

**Correct answer:** **3 times** — once per instance, all at the same time. This is almost never the desired behavior.

**Fix options:**
1. Use **ShedLock** (distributed lock)
2. Use **Quartz with JDBC JobStore** (cluster-aware)
3. Use a **dedicated scheduler instance** (only one instance runs scheduled tasks)
4. Implement application-level leadership election

---

### Trap 6: Cron 5-Field vs 6-Field

**Question:** "Will this work in Spring Boot?"

```java
@Scheduled(cron = "*/5 * * * *") // 5-field (Unix style)
```

**Correct answer:** **No.** Spring's `@Scheduled` requires a **6-field** cron expression with seconds as the first field. This will throw an `IllegalStateException` at startup.

**Fix:**
```java
@Scheduled(cron = "0 */5 * * * *") // 6-field — every 5 minutes
```

---

### Trap 7: Task Not Running After Exception

**Question:** "Your scheduled task was running fine for weeks, then it threw a NullPointerException. Now it never runs. Why?"

**Correct answer:** By default, an uncaught exception in a `@Scheduled` task causes the task to be **permanently deregistered** from the scheduler. The exception is logged, but the task is never rescheduled.

**Fix options:**
1. Add `try-catch` in the method to swallow/handle exceptions
2. Configure a custom `ErrorHandler` on the `TaskScheduler`
3. Use `@Async` (unhandled exceptions in async tasks don't affect the scheduler)

---

### Trap 8: Spring Proxy and @Scheduled

```java
@Service
public class MyService {

    // ❌ @Scheduled on a private method — AOP proxy cannot intercept!
    @Scheduled(fixedRate = 5000)
    private void privateScheduledTask() { /* MAY not work */ }

    // ❌ Calling a @Scheduled method internally bypasses the proxy
    public void doSomething() {
        privateScheduledTask(); // Direct call, not through proxy
    }
}
```

> [!WARNING]
> `@Scheduled` methods must be `public` and should not be called directly within the class. The `ScheduledAnnotationBeanPostProcessor` registers the method directly (not via proxy interception), so private methods might work, but it's implementation-dependent and not guaranteed. Always use `public` for clarity and reliability.

---

### Trap 9: TimeUnit String in @Scheduled

```java
// ❌ WRONG — fixedRate is in milliseconds, not seconds!
@Scheduled(fixedRate = 5) // Fires 200 times per second!

// ✅ CORRECT — explicit milliseconds
@Scheduled(fixedRate = 5000) // Every 5 seconds

// ✅ BETTER — use TimeUnit constant (Java 11+ with Duration)
@Scheduled(fixedRate = 5, timeUnit = TimeUnit.SECONDS)
```

Spring Boot 2.2+ supports `timeUnit` parameter:
```java
@Scheduled(fixedRate = 5, timeUnit = TimeUnit.SECONDS)
@Scheduled(fixedDelay = 30, timeUnit = TimeUnit.MINUTES)
@Scheduled(initialDelay = 1, fixedRate = 10, timeUnit = TimeUnit.SECONDS)
```

---

## 12. 🔍 Deep Dive — Internal Mechanics

### 12.1 How @EnableScheduling Works

`@EnableScheduling` imports `SchedulingConfiguration`, which registers a `ScheduledAnnotationBeanPostProcessor` bean.

```
Application startup sequence:
1. Spring scans classpath, creates bean definitions
2. @EnableScheduling → registers ScheduledAnnotationBeanPostProcessor
3. ScheduledAnnotationBeanPostProcessor implements:
   - BeanPostProcessor: scans all beans for @Scheduled methods
   - SmartInitializingSingleton: schedules all found tasks after all beans initialized
4. For each @Scheduled method found:
   a. Resolves fixedRate/fixedDelay/cron
   b. Resolves property placeholders (${...}) and SpEL (#{...})
   c. Calls TaskScheduler.scheduleAtFixedRate() or scheduleWithFixedDelay() or schedule(Runnable, Trigger)
5. TaskScheduler holds ScheduledFuture references for lifecycle management
```

### 12.2 How ShedLock Prevents Duplicate Execution

```java
// ShedLock's AOP advice (simplified):
@Around("@annotation(schedulerLock)")
public Object around(ProceedingJoinPoint pjp, SchedulerLock schedulerLock) throws Throwable {
    LockConfiguration lockConfig = getLockConfiguration(schedulerLock);

    // Try to acquire the lock
    Optional<SimpleLock> lock = lockProvider.lock(lockConfig);

    if (lock.isEmpty()) {
        // Another instance holds the lock — skip execution
        log.debug("Not executing '{}' — lock not obtained", lockConfig.getName());
        return null;
    }

    try {
        return pjp.proceed(); // Execute the actual task
    } finally {
        // Release the lock (or keep it for lockAtLeastFor duration)
        lock.get().unlock();
    }
}
```

### 12.3 Quartz Job Execution Flow

```
Quartz execution flow:
1. QuartzScheduler starts → loads triggers from JobStore (DB or RAM)
2. Trigger fires at scheduled time → QuartzScheduler.notifySchedulerThread()
3. JobRunShell created for the job execution
4. Job instance obtained (via JobFactory — SpringBeanJobFactory for Spring)
5. JobExecutionContext built with merged JobDataMap
6. Job.execute(context) called
7. On success → trigger state updated in JobStore → next fire time calculated
8. On failure → misfire policy applied (fire once, ignore, etc.)
```

### 12.4 Chunk Processing in Spring Batch

```
Chunk processing (chunk size = 100):

Transaction 1:
  Reader: read items 1-100
  Processor: process each item
  Writer: write all 100 in one batch
  Commit transaction ← checkpoint!

Transaction 2:
  Reader: read items 101-200
  Processor: process each item
  Writer: write all 100 in one batch
  Commit transaction ← checkpoint!

On failure at item 175:
  Transaction 2 is rolled back (items 101-175 NOT written)
  On restart: Job resumes from item 101 (last commit point)
  Only processes items 101-200 again, not 1-100
```

---

## 13. 📝 Summary & Key Takeaways

| Topic | Key Takeaway |
|---|---|
| `@EnableScheduling` | Must opt-in; triggers `ScheduledAnnotationBeanPostProcessor` |
| `fixedRate` | Measured from task **start**; overlaps if task is slow (with `@Async`) |
| `fixedDelay` | Measured from task **end**; always waits full delay |
| `cron` | 6-field in Spring (seconds first); 5-field in Unix |
| Default threading | **1 thread** for ALL tasks — the #1 trap |
| Task failure | Uncaught exception → task **dies forever** (configure ErrorHandler) |
| Multi-instance | `@Scheduled` runs on ALL instances — use ShedLock or Quartz |
| ShedLock | DB-backed distributed lock; `lockAtMostFor` is the safety net |
| Quartz | Enterprise scheduler; persistent, cluster-aware, misfire-handling |
| Spring Batch | For large-scale batch processing; chunk-oriented with restart support |
| Graceful shutdown | Set `await-termination` to prevent mid-task kills |
| Idempotency | Design all tasks to be safely re-runnable |

### The Decision Tree

```
Need to run code periodically?
│
├── Single instance app?
│   ├── Simple timing (every N seconds/minutes) → @Scheduled + fixedRate/fixedDelay
│   ├── Calendar-based → @Scheduled + cron
│   └── Dynamic schedule → SchedulingConfigurer
│
└── Multi-instance app (k8s, multiple replicas)?
    ├── Task must run once across cluster?
    │   ├── Simple → @Scheduled + ShedLock
    │   └── Complex/persistent/clustered → Quartz with JDBC JobStore
    │
    └── Processing millions of records?
        └── Spring Batch (+ Quartz or @Scheduled for triggering)
```

---

## 14. ❓ Practice Questions

### Conceptual Questions

1. **What is the default thread pool size for `@Scheduled` tasks in Spring Boot? What are the consequences of this default?**

2. **Explain the difference between `fixedRate` and `fixedDelay`. Draw a timeline showing both when a task takes 7 seconds and the interval is 5 seconds.**

3. **Why does Spring's cron use 6 fields while Unix cron uses 5 fields? What is the extra field?**

4. **What happens to a `@Scheduled` task after it throws an unhandled `RuntimeException`? How do you prevent this?**

5. **You have a Spring Boot app deployed as 3 Kubernetes pods. Each pod has a `@Scheduled` task that runs every minute. Describe the problem and two solutions.**

6. **What is the purpose of `lockAtMostFor` in ShedLock? What problem does it solve?**

7. **Explain the difference between Quartz `SimpleTrigger` and `CronTrigger`.**

8. **What is chunk-oriented processing in Spring Batch? Why is it better than processing records one at a time?**

### Coding Questions

9. **Write a `@Scheduled` task that:**
   - Runs every 15 minutes between 8 AM and 6 PM on weekdays
   - Is configured via application properties
   - Has proper error handling that doesn't kill the task
   - Logs execution time and reports metrics

10. **Configure a Spring Boot application with:**
    - A multi-threaded `TaskScheduler` with 5 threads
    - A custom `ErrorHandler` that sends alerts
    - A `@Scheduled` task using `@Async` to avoid blocking

11. **Implement a `SchedulingConfigurer` that reads cron expressions from a database and allows runtime updates via a REST endpoint.**

12. **Write a complete Quartz job that:**
    - Processes a queue of pending emails
    - Uses `JobDataMap` to pass configuration
    - Handles failures with retry logic
    - Is configured with a CronTrigger for 8 AM daily

13. **Design a distributed scheduled task system using ShedLock that:**
    - Runs a cleanup job every night at 2 AM
    - Ensures only one instance runs it
    - Has a failsafe if the running instance crashes
    - Includes monitoring/alerting

### Scenario Questions

14. **Your nightly report generation job was running fine but stopped after a database timeout 3 weeks ago. No one noticed because no alerts were set up. What would you change in the design?**

15. **You need to process 10 million user records nightly, updating their loyalty tier based on purchase history. Each record requires a lookup + update. `@Scheduled` + direct DB update is too slow. Design the solution.**

16. **Your application is deployed globally across 3 regions (US, EU, APAC). Each region has 5 instances. A report must be generated once per region per day at 8 AM local time. Design the scheduling solution.**

---

*Module 07 — Scheduling & Background Jobs | Spring Boot Deep Dive Series*
*Spring Boot 3.x | Spring 6.x | Quartz 2.3.x | ShedLock 5.x | Spring Batch 5.x*
