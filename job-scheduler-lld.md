# Job Scheduler System - Low Level Design

## Problem Statement

Design a low-level system for a Job Scheduler that allows creating, persisting, dispatching, and executing jobs with support for different scheduling strategies, priorities, retries, and concurrent execution.

### Functional Requirements
1. Create and schedule jobs (immediate, delayed, recurring/cron)
2. Persist jobs to ensure durability across system restarts
3. Dispatch jobs to worker threads for execution
4. Execute jobs with configurable concurrency
5. Support job priorities
6. Handle job failures with retry mechanism
7. Track job status and execution history
8. Cancel scheduled jobs
9. Query job status and results
10. Support different job types with custom logic

### Non-Functional Requirements
- High availability and fault tolerance
- Scalable to handle thousands of jobs
- Low latency for job scheduling and execution
- Thread-safe concurrent operations
- Efficient resource utilization
- Monitoring and observability

---

## Core Entities & Classes

### Job Definition

```java
enum JobStatus {
    PENDING,      // Job created but not yet scheduled
    SCHEDULED,    // Job scheduled for execution
    RUNNING,      // Job currently executing
    COMPLETED,    // Job completed successfully
    FAILED,       // Job failed
    CANCELLED,    // Job cancelled by user
    RETRYING      // Job failed but will retry
}

enum JobPriority {
    LOW(1),
    NORMAL(5),
    HIGH(10),
    CRITICAL(20);

    private final int value;

    JobPriority(int value) {
        this.value = value;
    }

    public int getValue() {
        return value;
    }
}

enum JobType {
    ONE_TIME,     // Execute once
    RECURRING,    // Execute on schedule (cron)
    DELAYED       // Execute after delay
}

abstract class Job {
    protected final String jobId;
    protected final String name;
    protected final JobType type;
    protected final JobPriority priority;
    protected final long createdAt;
    protected long scheduledAt;
    protected long executedAt;
    protected JobStatus status;
    protected int retryCount;
    protected int maxRetries;
    protected String result;
    protected String errorMessage;
    protected Map<String, Object> metadata;

    public Job(String name, JobType type, JobPriority priority, int maxRetries) {
        this.jobId = UUID.randomUUID().toString();
        this.name = name;
        this.type = type;
        this.priority = priority;
        this.maxRetries = maxRetries;
        this.createdAt = System.currentTimeMillis();
        this.status = JobStatus.PENDING;
        this.retryCount = 0;
        this.metadata = new HashMap<>();
    }

    public abstract void execute() throws Exception;

    public String getJobId() { return jobId; }
    public String getName() { return name; }
    public JobType getType() { return type; }
    public JobPriority getPriority() { return priority; }
    public long getCreatedAt() { return createdAt; }
    public long getScheduledAt() { return scheduledAt; }
    public long getExecutedAt() { return executedAt; }
    public JobStatus getStatus() { return status; }
    public int getRetryCount() { return retryCount; }
    public int getMaxRetries() { return maxRetries; }
    public String getResult() { return result; }
    public String getErrorMessage() { return errorMessage; }
    public Map<String, Object> getMetadata() { return metadata; }

    public void setScheduledAt(long scheduledAt) { this.scheduledAt = scheduledAt; }
    public void setExecutedAt(long executedAt) { this.executedAt = executedAt; }
    public void setStatus(JobStatus status) { this.status = status; }
    public void setResult(String result) { this.result = result; }
    public void setErrorMessage(String errorMessage) { this.errorMessage = errorMessage; }
    public void incrementRetryCount() { this.retryCount++; }

    public boolean canRetry() {
        return retryCount < maxRetries;
    }

    @Override
    public String toString() {
        return String.format("Job{id='%s', name='%s', status=%s, priority=%s}",
            jobId, name, status, priority);
    }
}
```

### Concrete Job Implementations

```java
class EmailJob extends Job {
    private final String recipient;
    private final String subject;
    private final String body;

    public EmailJob(String name, String recipient, String subject, String body) {
        super(name, JobType.ONE_TIME, JobPriority.NORMAL, 3);
        this.recipient = recipient;
        this.subject = subject;
        this.body = body;
        metadata.put("recipient", recipient);
        metadata.put("subject", subject);
    }

    @Override
    public void execute() throws Exception {
        System.out.println("Sending email to: " + recipient);
        System.out.println("Subject: " + subject);
        System.out.println("Body: " + body);

        Thread.sleep(1000);

        setResult("Email sent successfully to " + recipient);
    }
}

class DataProcessingJob extends Job {
    private final String dataSource;
    private final int recordCount;

    public DataProcessingJob(String name, String dataSource, int recordCount) {
        super(name, JobType.ONE_TIME, JobPriority.HIGH, 2);
        this.dataSource = dataSource;
        this.recordCount = recordCount;
        metadata.put("dataSource", dataSource);
        metadata.put("recordCount", recordCount);
    }

    @Override
    public void execute() throws Exception {
        System.out.println("Processing data from: " + dataSource);
        System.out.println("Record count: " + recordCount);

        for (int i = 0; i < recordCount; i++) {
            Thread.sleep(100);
        }

        setResult("Processed " + recordCount + " records from " + dataSource);
    }
}

class ReportGenerationJob extends Job {
    private final String reportType;
    private final LocalDate startDate;
    private final LocalDate endDate;

    public ReportGenerationJob(String name, String reportType,
                               LocalDate startDate, LocalDate endDate) {
        super(name, JobType.ONE_TIME, JobPriority.NORMAL, 1);
        this.reportType = reportType;
        this.startDate = startDate;
        this.endDate = endDate;
        metadata.put("reportType", reportType);
        metadata.put("startDate", startDate.toString());
        metadata.put("endDate", endDate.toString());
    }

    @Override
    public void execute() throws Exception {
        System.out.println("Generating " + reportType + " report");
        System.out.println("Date range: " + startDate + " to " + endDate);

        Thread.sleep(2000);

        setResult("Report generated: " + reportType + "_" +
            startDate + "_to_" + endDate + ".pdf");
    }
}
```

### Scheduled Job

```java
class ScheduledJob implements Comparable<ScheduledJob> {
    private final Job job;
    private final long executionTime;
    private final String cronExpression;

    public ScheduledJob(Job job, long executionTime) {
        this.job = job;
        this.executionTime = executionTime;
        this.cronExpression = null;
    }

    public ScheduledJob(Job job, String cronExpression) {
        this.job = job;
        this.cronExpression = cronExpression;
        this.executionTime = calculateNextExecutionTime(cronExpression);
    }

    private long calculateNextExecutionTime(String cronExpression) {
        // Simplified cron parsing - in production use library like Quartz
        return System.currentTimeMillis() + 60000; // Default 1 minute
    }

    public Job getJob() {
        return job;
    }

    public long getExecutionTime() {
        return executionTime;
    }

    public String getCronExpression() {
        return cronExpression;
    }

    public boolean isRecurring() {
        return cronExpression != null;
    }

    public ScheduledJob getNextOccurrence() {
        if (!isRecurring()) {
            return null;
        }
        return new ScheduledJob(job, cronExpression);
    }

    @Override
    public int compareTo(ScheduledJob other) {
        int timeCompare = Long.compare(this.executionTime, other.executionTime);
        if (timeCompare != 0) {
            return timeCompare;
        }
        return Integer.compare(
            other.job.getPriority().getValue(),
            this.job.getPriority().getValue()
        );
    }
}
```

### Job Execution Context

```java
class JobExecutionContext {
    private final String executionId;
    private final Job job;
    private final long startTime;
    private long endTime;
    private boolean success;
    private String errorDetails;

    public JobExecutionContext(Job job) {
        this.executionId = UUID.randomUUID().toString();
        this.job = job;
        this.startTime = System.currentTimeMillis();
    }

    public void markCompleted() {
        this.endTime = System.currentTimeMillis();
        this.success = true;
    }

    public void markFailed(String errorDetails) {
        this.endTime = System.currentTimeMillis();
        this.success = false;
        this.errorDetails = errorDetails;
    }

    public String getExecutionId() { return executionId; }
    public Job getJob() { return job; }
    public long getStartTime() { return startTime; }
    public long getEndTime() { return endTime; }
    public long getDuration() { return endTime - startTime; }
    public boolean isSuccess() { return success; }
    public String getErrorDetails() { return errorDetails; }
}
```

---

## Persistence Layer

### Job Repository Interface

```java
interface JobRepository {
    void save(Job job);

    void update(Job job);

    Optional<Job> findById(String jobId);

    List<Job> findByStatus(JobStatus status);

    List<Job> findAll();

    void delete(String jobId);

    List<JobExecutionContext> getExecutionHistory(String jobId);

    void saveExecutionContext(JobExecutionContext context);
}
```

### In-Memory Implementation

```java
class InMemoryJobRepository implements JobRepository {
    private final Map<String, Job> jobs = new ConcurrentHashMap<>();
    private final Map<String, List<JobExecutionContext>> executionHistory =
        new ConcurrentHashMap<>();

    @Override
    public void save(Job job) {
        jobs.put(job.getJobId(), job);
    }

    @Override
    public void update(Job job) {
        jobs.put(job.getJobId(), job);
    }

    @Override
    public Optional<Job> findById(String jobId) {
        return Optional.ofNullable(jobs.get(jobId));
    }

    @Override
    public List<Job> findByStatus(JobStatus status) {
        return jobs.values().stream()
            .filter(job -> job.getStatus() == status)
            .collect(Collectors.toList());
    }

    @Override
    public List<Job> findAll() {
        return new ArrayList<>(jobs.values());
    }

    @Override
    public void delete(String jobId) {
        jobs.remove(jobId);
        executionHistory.remove(jobId);
    }

    @Override
    public List<JobExecutionContext> getExecutionHistory(String jobId) {
        return executionHistory.getOrDefault(jobId, new ArrayList<>());
    }

    @Override
    public void saveExecutionContext(JobExecutionContext context) {
        String jobId = context.getJob().getJobId();
        executionHistory.computeIfAbsent(jobId, k -> new ArrayList<>())
            .add(context);
    }
}
```

### Database Implementation (Skeleton)

```java
class DatabaseJobRepository implements JobRepository {
    private final DataSource dataSource;

    public DatabaseJobRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public void save(Job job) {
        String sql = "INSERT INTO jobs (job_id, name, type, priority, status, " +
                    "created_at, max_retries, metadata) VALUES (?, ?, ?, ?, ?, ?, ?, ?)";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, job.getJobId());
            stmt.setString(2, job.getName());
            stmt.setString(3, job.getType().name());
            stmt.setString(4, job.getPriority().name());
            stmt.setString(5, job.getStatus().name());
            stmt.setLong(6, job.getCreatedAt());
            stmt.setInt(7, job.getMaxRetries());
            stmt.setString(8, serializeMetadata(job.getMetadata()));

            stmt.executeUpdate();

        } catch (SQLException e) {
            throw new RuntimeException("Failed to save job", e);
        }
    }

    @Override
    public void update(Job job) {
        String sql = "UPDATE jobs SET status = ?, retry_count = ?, " +
                    "executed_at = ?, result = ?, error_message = ? WHERE job_id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, job.getStatus().name());
            stmt.setInt(2, job.getRetryCount());
            stmt.setLong(3, job.getExecutedAt());
            stmt.setString(4, job.getResult());
            stmt.setString(5, job.getErrorMessage());
            stmt.setString(6, job.getJobId());

            stmt.executeUpdate();

        } catch (SQLException e) {
            throw new RuntimeException("Failed to update job", e);
        }
    }

    @Override
    public Optional<Job> findById(String jobId) {
        // Implementation to fetch from database and reconstruct Job object
        return Optional.empty();
    }

    @Override
    public List<Job> findByStatus(JobStatus status) {
        // Implementation to query jobs by status
        return new ArrayList<>();
    }

    @Override
    public List<Job> findAll() {
        // Implementation to fetch all jobs
        return new ArrayList<>();
    }

    @Override
    public void delete(String jobId) {
        String sql = "DELETE FROM jobs WHERE job_id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, jobId);
            stmt.executeUpdate();

        } catch (SQLException e) {
            throw new RuntimeException("Failed to delete job", e);
        }
    }

    @Override
    public List<JobExecutionContext> getExecutionHistory(String jobId) {
        // Implementation to fetch execution history
        return new ArrayList<>();
    }

    @Override
    public void saveExecutionContext(JobExecutionContext context) {
        String sql = "INSERT INTO job_executions (execution_id, job_id, start_time, " +
                    "end_time, success, error_details) VALUES (?, ?, ?, ?, ?, ?)";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, context.getExecutionId());
            stmt.setString(2, context.getJob().getJobId());
            stmt.setLong(3, context.getStartTime());
            stmt.setLong(4, context.getEndTime());
            stmt.setBoolean(5, context.isSuccess());
            stmt.setString(6, context.getErrorDetails());

            stmt.executeUpdate();

        } catch (SQLException e) {
            throw new RuntimeException("Failed to save execution context", e);
        }
    }

    private String serializeMetadata(Map<String, Object> metadata) {
        // Convert metadata to JSON string
        return "{}"; // Placeholder
    }
}
```

---

## Job Dispatcher

```java
interface JobDispatcher {
    void dispatch(Job job);

    void start();

    void shutdown();
}

class PriorityQueueJobDispatcher implements JobDispatcher {
    private final PriorityBlockingQueue<ScheduledJob> jobQueue;
    private final JobExecutor executor;
    private final JobRepository repository;
    private final Thread dispatcherThread;
    private volatile boolean running;

    public PriorityQueueJobDispatcher(JobExecutor executor, JobRepository repository) {
        this.jobQueue = new PriorityBlockingQueue<>();
        this.executor = executor;
        this.repository = repository;
        this.dispatcherThread = new Thread(this::dispatchLoop, "JobDispatcher");
        this.running = false;
    }

    @Override
    public void dispatch(Job job) {
        ScheduledJob scheduledJob = new ScheduledJob(job, job.getScheduledAt());
        job.setStatus(JobStatus.SCHEDULED);
        repository.update(job);
        jobQueue.offer(scheduledJob);
    }

    @Override
    public void start() {
        running = true;
        dispatcherThread.start();
    }

    @Override
    public void shutdown() {
        running = false;
        dispatcherThread.interrupt();
        try {
            dispatcherThread.join(5000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void dispatchLoop() {
        while (running) {
            try {
                ScheduledJob scheduledJob = jobQueue.poll(1, TimeUnit.SECONDS);

                if (scheduledJob == null) {
                    continue;
                }

                long currentTime = System.currentTimeMillis();
                long executionTime = scheduledJob.getExecutionTime();

                if (executionTime > currentTime) {
                    long delay = executionTime - currentTime;
                    Thread.sleep(Math.min(delay, 1000));
                    jobQueue.offer(scheduledJob);
                    continue;
                }

                Job job = scheduledJob.getJob();
                executor.submit(job);

                if (scheduledJob.isRecurring()) {
                    ScheduledJob nextOccurrence = scheduledJob.getNextOccurrence();
                    if (nextOccurrence != null) {
                        jobQueue.offer(nextOccurrence);
                    }
                }

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                System.err.println("Error in dispatcher loop: " + e.getMessage());
            }
        }
    }

    public int getQueueSize() {
        return jobQueue.size();
    }
}
```

---

## Job Executor

```java
interface JobExecutor {
    void submit(Job job);

    void start();

    void shutdown();

    JobExecutionStats getStats();
}

class ThreadPoolJobExecutor implements JobExecutor {
    private final ExecutorService executorService;
    private final JobRepository repository;
    private final RetryPolicy retryPolicy;
    private final JobExecutionStats stats;
    private volatile boolean running;

    public ThreadPoolJobExecutor(int threadPoolSize,
                                JobRepository repository,
                                RetryPolicy retryPolicy) {
        this.executorService = Executors.newFixedThreadPool(threadPoolSize);
        this.repository = repository;
        this.retryPolicy = retryPolicy;
        this.stats = new JobExecutionStats();
        this.running = false;
    }

    @Override
    public void submit(Job job) {
        if (!running) {
            throw new IllegalStateException("Executor is not running");
        }

        executorService.submit(() -> executeJob(job));
    }

    @Override
    public void start() {
        running = true;
    }

    @Override
    public void shutdown() {
        running = false;
        executorService.shutdown();
        try {
            if (!executorService.awaitTermination(60, TimeUnit.SECONDS)) {
                executorService.shutdownNow();
            }
        } catch (InterruptedException e) {
            executorService.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    @Override
    public JobExecutionStats getStats() {
        return stats;
    }

    private void executeJob(Job job) {
        JobExecutionContext context = new JobExecutionContext(job);

        try {
            job.setStatus(JobStatus.RUNNING);
            job.setExecutedAt(System.currentTimeMillis());
            repository.update(job);

            stats.incrementRunning();

            job.execute();

            job.setStatus(JobStatus.COMPLETED);
            repository.update(job);

            context.markCompleted();
            stats.incrementCompleted();

        } catch (Exception e) {
            handleJobFailure(job, context, e);

        } finally {
            stats.decrementRunning();
            repository.saveExecutionContext(context);
        }
    }

    private void handleJobFailure(Job job, JobExecutionContext context, Exception e) {
        String errorMessage = e.getMessage();
        job.setErrorMessage(errorMessage);
        context.markFailed(errorMessage);

        if (job.canRetry()) {
            job.incrementRetryCount();
            job.setStatus(JobStatus.RETRYING);
            repository.update(job);

            long retryDelay = retryPolicy.getRetryDelay(job.getRetryCount());
            scheduleRetry(job, retryDelay);

            stats.incrementRetried();
        } else {
            job.setStatus(JobStatus.FAILED);
            repository.update(job);
            stats.incrementFailed();
        }
    }

    private void scheduleRetry(Job job, long delayMs) {
        new Thread(() -> {
            try {
                Thread.sleep(delayMs);
                submit(job);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }
}

class JobExecutionStats {
    private final AtomicLong totalSubmitted = new AtomicLong(0);
    private final AtomicLong totalCompleted = new AtomicLong(0);
    private final AtomicLong totalFailed = new AtomicLong(0);
    private final AtomicLong totalRetried = new AtomicLong(0);
    private final AtomicInteger currentlyRunning = new AtomicInteger(0);

    public void incrementRunning() {
        totalSubmitted.incrementAndGet();
        currentlyRunning.incrementAndGet();
    }

    public void decrementRunning() {
        currentlyRunning.decrementAndGet();
    }

    public void incrementCompleted() {
        totalCompleted.incrementAndGet();
    }

    public void incrementFailed() {
        totalFailed.incrementAndGet();
    }

    public void incrementRetried() {
        totalRetried.incrementAndGet();
    }

    public long getTotalSubmitted() { return totalSubmitted.get(); }
    public long getTotalCompleted() { return totalCompleted.get(); }
    public long getTotalFailed() { return totalFailed.get(); }
    public long getTotalRetried() { return totalRetried.get(); }
    public int getCurrentlyRunning() { return currentlyRunning.get(); }

    @Override
    public String toString() {
        return String.format(
            "JobExecutionStats{submitted=%d, completed=%d, failed=%d, retried=%d, running=%d}",
            getTotalSubmitted(), getTotalCompleted(), getTotalFailed(),
            getTotalRetried(), getCurrentlyRunning()
        );
    }
}
```

---

## Retry Policy

```java
interface RetryPolicy {
    long getRetryDelay(int attemptNumber);
}

class ExponentialBackoffRetryPolicy implements RetryPolicy {
    private final long baseDelayMs;
    private final long maxDelayMs;
    private final double multiplier;

    public ExponentialBackoffRetryPolicy(long baseDelayMs, long maxDelayMs, double multiplier) {
        this.baseDelayMs = baseDelayMs;
        this.maxDelayMs = maxDelayMs;
        this.multiplier = multiplier;
    }

    @Override
    public long getRetryDelay(int attemptNumber) {
        long delay = (long) (baseDelayMs * Math.pow(multiplier, attemptNumber - 1));
        return Math.min(delay, maxDelayMs);
    }
}

class FixedDelayRetryPolicy implements RetryPolicy {
    private final long delayMs;

    public FixedDelayRetryPolicy(long delayMs) {
        this.delayMs = delayMs;
    }

    @Override
    public long getRetryDelay(int attemptNumber) {
        return delayMs;
    }
}
```

---

## Job Scheduler Service

```java
interface JobScheduler {
    String scheduleJob(Job job);

    String scheduleDelayedJob(Job job, long delayMs);

    String scheduleRecurringJob(Job job, String cronExpression);

    boolean cancelJob(String jobId);

    Optional<Job> getJob(String jobId);

    List<Job> getJobsByStatus(JobStatus status);

    void start();

    void shutdown();
}

class JobSchedulerImpl implements JobScheduler {
    private final JobRepository repository;
    private final JobDispatcher dispatcher;
    private final JobExecutor executor;

    public JobSchedulerImpl(JobRepository repository,
                           JobDispatcher dispatcher,
                           JobExecutor executor) {
        this.repository = repository;
        this.dispatcher = dispatcher;
        this.executor = executor;
    }

    @Override
    public String scheduleJob(Job job) {
        job.setScheduledAt(System.currentTimeMillis());
        repository.save(job);
        dispatcher.dispatch(job);
        return job.getJobId();
    }

    @Override
    public String scheduleDelayedJob(Job job, long delayMs) {
        long scheduledTime = System.currentTimeMillis() + delayMs;
        job.setScheduledAt(scheduledTime);
        repository.save(job);
        dispatcher.dispatch(job);
        return job.getJobId();
    }

    @Override
    public String scheduleRecurringJob(Job job, String cronExpression) {
        job.setScheduledAt(System.currentTimeMillis());
        repository.save(job);

        ScheduledJob scheduledJob = new ScheduledJob(job, cronExpression);
        job.setStatus(JobStatus.SCHEDULED);
        repository.update(job);

        if (dispatcher instanceof PriorityQueueJobDispatcher) {
            PriorityQueueJobDispatcher pqDispatcher = (PriorityQueueJobDispatcher) dispatcher;
            pqDispatcher.dispatch(job);
        }

        return job.getJobId();
    }

    @Override
    public boolean cancelJob(String jobId) {
        Optional<Job> jobOpt = repository.findById(jobId);

        if (jobOpt.isPresent()) {
            Job job = jobOpt.get();

            if (job.getStatus() == JobStatus.RUNNING) {
                return false;
            }

            job.setStatus(JobStatus.CANCELLED);
            repository.update(job);
            return true;
        }

        return false;
    }

    @Override
    public Optional<Job> getJob(String jobId) {
        return repository.findById(jobId);
    }

    @Override
    public List<Job> getJobsByStatus(JobStatus status) {
        return repository.findByStatus(status);
    }

    @Override
    public void start() {
        executor.start();
        dispatcher.start();

        loadPendingJobs();
    }

    @Override
    public void shutdown() {
        dispatcher.shutdown();
        executor.shutdown();
    }

    private void loadPendingJobs() {
        List<Job> pendingJobs = repository.findByStatus(JobStatus.SCHEDULED);
        for (Job job : pendingJobs) {
            dispatcher.dispatch(job);
        }
    }
}
```

---

## Configuration

```java
class JobSchedulerConfig {
    private final int executorThreadPoolSize;
    private final long baseRetryDelayMs;
    private final long maxRetryDelayMs;
    private final double retryMultiplier;

    private JobSchedulerConfig(Builder builder) {
        this.executorThreadPoolSize = builder.executorThreadPoolSize;
        this.baseRetryDelayMs = builder.baseRetryDelayMs;
        this.maxRetryDelayMs = builder.maxRetryDelayMs;
        this.retryMultiplier = builder.retryMultiplier;
    }

    public static class Builder {
        private int executorThreadPoolSize = 10;
        private long baseRetryDelayMs = 1000;
        private long maxRetryDelayMs = 60000;
        private double retryMultiplier = 2.0;

        public Builder executorThreadPoolSize(int size) {
            this.executorThreadPoolSize = size;
            return this;
        }

        public Builder baseRetryDelayMs(long delayMs) {
            this.baseRetryDelayMs = delayMs;
            return this;
        }

        public Builder maxRetryDelayMs(long delayMs) {
            this.maxRetryDelayMs = delayMs;
            return this;
        }

        public Builder retryMultiplier(double multiplier) {
            this.retryMultiplier = multiplier;
            return this;
        }

        public JobSchedulerConfig build() {
            return new JobSchedulerConfig(this);
        }
    }

    public int getExecutorThreadPoolSize() { return executorThreadPoolSize; }
    public long getBaseRetryDelayMs() { return baseRetryDelayMs; }
    public long getMaxRetryDelayMs() { return maxRetryDelayMs; }
    public double getRetryMultiplier() { return retryMultiplier; }
}
```

---

## Factory

```java
class JobSchedulerFactory {

    public static JobScheduler createScheduler(JobSchedulerConfig config) {
        JobRepository repository = new InMemoryJobRepository();

        RetryPolicy retryPolicy = new ExponentialBackoffRetryPolicy(
            config.getBaseRetryDelayMs(),
            config.getMaxRetryDelayMs(),
            config.getRetryMultiplier()
        );

        JobExecutor executor = new ThreadPoolJobExecutor(
            config.getExecutorThreadPoolSize(),
            repository,
            retryPolicy
        );

        JobDispatcher dispatcher = new PriorityQueueJobDispatcher(executor, repository);

        return new JobSchedulerImpl(repository, dispatcher, executor);
    }

    public static JobScheduler createSchedulerWithDatabase(
            JobSchedulerConfig config,
            DataSource dataSource) {

        JobRepository repository = new DatabaseJobRepository(dataSource);

        RetryPolicy retryPolicy = new ExponentialBackoffRetryPolicy(
            config.getBaseRetryDelayMs(),
            config.getMaxRetryDelayMs(),
            config.getRetryMultiplier()
        );

        JobExecutor executor = new ThreadPoolJobExecutor(
            config.getExecutorThreadPoolSize(),
            repository,
            retryPolicy
        );

        JobDispatcher dispatcher = new PriorityQueueJobDispatcher(executor, repository);

        return new JobSchedulerImpl(repository, dispatcher, executor);
    }
}
```

---

## Demo/Driver Code

```java
class JobSchedulerDemo {

    public static void main(String[] args) throws Exception {
        System.out.println("=== Job Scheduler System Demo ===\n");

        JobSchedulerConfig config = new JobSchedulerConfig.Builder()
            .executorThreadPoolSize(5)
            .baseRetryDelayMs(2000)
            .maxRetryDelayMs(30000)
            .retryMultiplier(2.0)
            .build();

        JobScheduler scheduler = JobSchedulerFactory.createScheduler(config);
        scheduler.start();

        try {
            demonstrateImmediateJobExecution(scheduler);
            Thread.sleep(2000);

            demonstrateDelayedJobExecution(scheduler);
            Thread.sleep(3000);

            demonstratePriorityJobs(scheduler);
            Thread.sleep(2000);

            demonstrateJobCancellation(scheduler);
            Thread.sleep(2000);

            demonstrateJobRetry(scheduler);
            Thread.sleep(10000);

            demonstrateJobStatusQuery(scheduler);

        } finally {
            System.out.println("\n=== Shutting down scheduler ===");
            scheduler.shutdown();
            System.out.println("Scheduler shutdown complete");
        }
    }

    private static void demonstrateImmediateJobExecution(JobScheduler scheduler) {
        System.out.println("1. Immediate Job Execution");

        EmailJob emailJob = new EmailJob(
            "Welcome Email",
            "user@example.com",
            "Welcome to our service!",
            "Thank you for signing up."
        );

        String jobId = scheduler.scheduleJob(emailJob);
        System.out.println("   ✓ Scheduled email job: " + jobId + "\n");
    }

    private static void demonstrateDelayedJobExecution(JobScheduler scheduler) {
        System.out.println("2. Delayed Job Execution");

        DataProcessingJob processingJob = new DataProcessingJob(
            "Daily Data Processing",
            "sales_database",
            5
        );

        String jobId = scheduler.scheduleDelayedJob(processingJob, 5000);
        System.out.println("   ✓ Scheduled delayed job (5s delay): " + jobId + "\n");
    }

    private static void demonstratePriorityJobs(JobScheduler scheduler) {
        System.out.println("3. Priority-Based Job Execution");

        Job lowPriorityJob = new ReportGenerationJob(
            "Monthly Report",
            "Sales Report",
            LocalDate.now().minusMonths(1),
            LocalDate.now()
        );

        Job highPriorityJob = new DataProcessingJob(
            "Critical Data Processing",
            "production_database",
            3
        );

        String lowId = scheduler.scheduleJob(lowPriorityJob);
        String highId = scheduler.scheduleJob(highPriorityJob);

        System.out.println("   ✓ Scheduled low priority job: " + lowId);
        System.out.println("   ✓ Scheduled high priority job: " + highId);
        System.out.println("   High priority job should execute first\n");
    }

    private static void demonstrateJobCancellation(JobScheduler scheduler) {
        System.out.println("4. Job Cancellation");

        EmailJob emailJob = new EmailJob(
            "Newsletter",
            "subscribers@example.com",
            "Monthly Newsletter",
            "Here's what's new this month..."
        );

        String jobId = scheduler.scheduleDelayedJob(emailJob, 60000);
        System.out.println("   ✓ Scheduled job: " + jobId);

        boolean cancelled = scheduler.cancelJob(jobId);
        System.out.println("   ✓ Job cancelled: " + cancelled + "\n");
    }

    private static void demonstrateJobRetry(JobScheduler scheduler) {
        System.out.println("5. Job Retry Mechanism");

        Job failingJob = new FailingJob("Failing Job Test", 2);
        String jobId = scheduler.scheduleJob(failingJob);

        System.out.println("   ✓ Scheduled job that will fail and retry: " + jobId);
        System.out.println("   Job will retry with exponential backoff\n");
    }

    private static void demonstrateJobStatusQuery(JobScheduler scheduler) {
        System.out.println("6. Job Status Query");

        List<Job> completedJobs = scheduler.getJobsByStatus(JobStatus.COMPLETED);
        List<Job> failedJobs = scheduler.getJobsByStatus(JobStatus.FAILED);
        List<Job> runningJobs = scheduler.getJobsByStatus(JobStatus.RUNNING);

        System.out.println("   Completed jobs: " + completedJobs.size());
        System.out.println("   Failed jobs: " + failedJobs.size());
        System.out.println("   Running jobs: " + runningJobs.size());

        System.out.println("\n   Job Details:");
        for (Job job : completedJobs) {
            System.out.println("   - " + job.getName() + ": " + job.getResult());
        }
        System.out.println();
    }

    static class FailingJob extends Job {
        private final int successAfterAttempts;

        public FailingJob(String name, int successAfterAttempts) {
            super(name, JobType.ONE_TIME, JobPriority.NORMAL, 3);
            this.successAfterAttempts = successAfterAttempts;
        }

        @Override
        public void execute() throws Exception {
            System.out.println("   Executing " + getName() + " (attempt " +
                (getRetryCount() + 1) + ")");

            if (getRetryCount() < successAfterAttempts) {
                throw new Exception("Simulated failure on attempt " + (getRetryCount() + 1));
            }

            setResult("Successfully completed after " + (getRetryCount() + 1) + " attempts");
        }
    }
}
```

---

## Key Design Considerations

### 1. Job Persistence

**Why it matters:**
- System recovery after crashes
- Audit trail and debugging
- Job history tracking

**Implementation:**
```java
// Save on creation
repository.save(job);

// Update on state changes
job.setStatus(JobStatus.RUNNING);
repository.update(job);

// Load pending jobs on startup
loadPendingJobs();
```

### 2. Priority-Based Scheduling

**PriorityBlockingQueue with custom comparator:**
```java
@Override
public int compareTo(ScheduledJob other) {
    // First by execution time
    int timeCompare = Long.compare(this.executionTime, other.executionTime);
    if (timeCompare != 0) return timeCompare;

    // Then by priority (higher first)
    return Integer.compare(
        other.job.getPriority().getValue(),
        this.job.getPriority().getValue()
    );
}
```

### 3. Execution Flow

```
Job Creation → Persistence → Scheduling → Dispatching → Execution → Completion
     ↓              ↓             ↓            ↓            ↓           ↓
  PENDING      Save to DB    SCHEDULED    Add to Queue   RUNNING   COMPLETED
                                                            ↓
                                                         FAILED
                                                            ↓
                                                        Retry?
                                                       ↙      ↘
                                                  RETRYING   FAILED
```

### 4. Thread Safety

**Multiple synchronization points:**
- `ConcurrentHashMap` for job storage
- `PriorityBlockingQueue` for job queue
- `AtomicInteger/AtomicLong` for stats
- Thread pool for concurrent execution

### 5. Retry Strategy

**Exponential backoff:**
```
Attempt 1: 1s delay
Attempt 2: 2s delay
Attempt 3: 4s delay
Attempt 4: 8s delay (capped at maxDelay)
```

**Benefits:**
- Reduces load on failing systems
- Increases chance of recovery
- Prevents thundering herd

### 6. Resource Management

**Bounded thread pool:**
- Prevents resource exhaustion
- Configurable concurrency level
- Graceful shutdown

---

## Advanced Features

### 1. Job Dependency Management

```java
class DependentJob extends Job {
    private final List<String> dependencyJobIds;

    public DependentJob(String name, List<String> dependencyJobIds) {
        super(name, JobType.ONE_TIME, JobPriority.NORMAL, 3);
        this.dependencyJobIds = dependencyJobIds;
    }

    public boolean areDependenciesMet(JobRepository repository) {
        for (String jobId : dependencyJobIds) {
            Optional<Job> depJob = repository.findById(jobId);
            if (!depJob.isPresent() || depJob.get().getStatus() != JobStatus.COMPLETED) {
                return false;
            }
        }
        return true;
    }

    @Override
    public void execute() throws Exception {
        // Execute after all dependencies are completed
    }
}
```

### 2. Job Chaining

```java
interface JobChain {
    void addJob(Job job);
    void execute();
}

class SequentialJobChain implements JobChain {
    private final List<Job> jobs = new ArrayList<>();
    private final JobScheduler scheduler;

    public SequentialJobChain(JobScheduler scheduler) {
        this.scheduler = scheduler;
    }

    @Override
    public void addJob(Job job) {
        jobs.add(job);
    }

    @Override
    public void execute() {
        for (int i = 0; i < jobs.size(); i++) {
            Job currentJob = jobs.get(i);

            if (i > 0) {
                Job previousJob = jobs.get(i - 1);
                // Wait for previous job to complete
                waitForCompletion(previousJob);
            }

            scheduler.scheduleJob(currentJob);
        }
    }

    private void waitForCompletion(Job job) {
        // Implementation to wait for job completion
    }
}
```

### 3. Dead Letter Queue

```java
class DeadLetterQueue {
    private final Queue<Job> failedJobs = new ConcurrentLinkedQueue<>();
    private final JobRepository repository;

    public void addFailedJob(Job job) {
        failedJobs.offer(job);
        job.setStatus(JobStatus.FAILED);
        repository.update(job);
    }

    public List<Job> getFailedJobs() {
        return new ArrayList<>(failedJobs);
    }

    public void retryFailedJob(String jobId, JobScheduler scheduler) {
        Job job = failedJobs.stream()
            .filter(j -> j.getJobId().equals(jobId))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Job not found"));

        failedJobs.remove(job);
        job.setStatus(JobStatus.PENDING);
        job.setErrorMessage(null);
        scheduler.scheduleJob(job);
    }
}
```

### 4. Job Monitoring

```java
class JobMonitor {
    private final JobScheduler scheduler;
    private final ScheduledExecutorService monitoringExecutor;

    public JobMonitor(JobScheduler scheduler) {
        this.scheduler = scheduler;
        this.monitoringExecutor = Executors.newSingleThreadScheduledExecutor();
    }

    public void startMonitoring(long intervalMs) {
        monitoringExecutor.scheduleAtFixedRate(
            this::checkJobHealth,
            intervalMs,
            intervalMs,
            TimeUnit.MILLISECONDS
        );
    }

    private void checkJobHealth() {
        List<Job> runningJobs = scheduler.getJobsByStatus(JobStatus.RUNNING);
        long currentTime = System.currentTimeMillis();

        for (Job job : runningJobs) {
            long runningTime = currentTime - job.getExecutedAt();

            if (runningTime > 300000) {
                System.err.println("WARNING: Job " + job.getJobId() +
                    " has been running for " + runningTime + "ms");
            }
        }
    }

    public void shutdown() {
        monitoringExecutor.shutdown();
    }
}
```

### 5. Distributed Job Scheduler (Sketch)

```java
class DistributedJobScheduler implements JobScheduler {
    private final JobScheduler localScheduler;
    private final DistributedLock distributedLock;
    private final String nodeId;

    @Override
    public String scheduleJob(Job job) {
        String lockKey = "job_schedule_lock";

        try {
            if (distributedLock.tryLock(lockKey, 5, TimeUnit.SECONDS)) {
                job.getMetadata().put("scheduledBy", nodeId);
                return localScheduler.scheduleJob(job);
            } else {
                throw new RuntimeException("Could not acquire lock for job scheduling");
            }
        } finally {
            distributedLock.unlock(lockKey);
        }
    }
}

interface DistributedLock {
    boolean tryLock(String key, long timeout, TimeUnit unit);
    void unlock(String key);
}
```

### 6. Cron Expression Support

```java
class CronExpression {
    private final String expression;

    public CronExpression(String expression) {
        this.expression = expression;
        validate();
    }

    public long getNextExecutionTime(long currentTime) {
        // Parse cron expression and calculate next execution
        // Format: "0 0 12 * * ?" (Every day at 12:00 PM)

        // Simple implementation - use Quartz CronExpression in production
        return currentTime + 86400000; // Next day
    }

    private void validate() {
        // Validate cron expression format
        String[] parts = expression.split(" ");
        if (parts.length != 6) {
            throw new IllegalArgumentException("Invalid cron expression");
        }
    }
}
```

---

## Database Schema

```sql
CREATE TABLE jobs (
    job_id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    priority VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at BIGINT NOT NULL,
    scheduled_at BIGINT,
    executed_at BIGINT,
    max_retries INT NOT NULL,
    retry_count INT DEFAULT 0,
    result TEXT,
    error_message TEXT,
    metadata TEXT,
    cron_expression VARCHAR(100),
    INDEX idx_status (status),
    INDEX idx_scheduled_at (scheduled_at),
    INDEX idx_created_at (created_at)
);

CREATE TABLE job_executions (
    execution_id VARCHAR(36) PRIMARY KEY,
    job_id VARCHAR(36) NOT NULL,
    start_time BIGINT NOT NULL,
    end_time BIGINT,
    success BOOLEAN NOT NULL,
    error_details TEXT,
    FOREIGN KEY (job_id) REFERENCES jobs(job_id) ON DELETE CASCADE,
    INDEX idx_job_id (job_id),
    INDEX idx_start_time (start_time)
);

CREATE TABLE job_dependencies (
    job_id VARCHAR(36) NOT NULL,
    dependency_job_id VARCHAR(36) NOT NULL,
    PRIMARY KEY (job_id, dependency_job_id),
    FOREIGN KEY (job_id) REFERENCES jobs(job_id) ON DELETE CASCADE,
    FOREIGN KEY (dependency_job_id) REFERENCES jobs(job_id) ON DELETE CASCADE
);
```

---

## Class Diagram

```
┌─────────────────────┐
│       Job           │
│   <<abstract>>      │
├─────────────────────┤
│ - jobId             │
│ - name              │
│ - type              │
│ - priority          │
│ - status            │
├─────────────────────┤
│ + execute()         │
└─────────────────────┘
         △
         │
    ┌────┴─────┬──────────┐
    │          │          │
┌─────────┐ ┌─────────┐ ┌─────────┐
│EmailJob │ │DataProc │ │ReportJob│
└─────────┘ └─────────┘ └─────────┘

┌──────────────────────┐         ┌──────────────────────┐
│   JobScheduler       │────────▷│   JobRepository      │
├──────────────────────┤         ├──────────────────────┤
│ + scheduleJob()      │         │ + save()             │
│ + cancelJob()        │         │ + findById()         │
│ + getJob()           │         │ + update()           │
└──────────────────────┘         └──────────────────────┘
         │
         │ uses
         ▽
┌──────────────────────┐         ┌──────────────────────┐
│   JobDispatcher      │────────▷│    JobExecutor       │
├──────────────────────┤         ├──────────────────────┤
│ - jobQueue           │         │ - executorService    │
│ + dispatch()         │         │ + submit()           │
│ + start()            │         │ + executeJob()       │
└──────────────────────┘         └──────────────────────┘
                                          │
                                          │ uses
                                          ▽
                                 ┌──────────────────────┐
                                 │   RetryPolicy        │
                                 ├──────────────────────┤
                                 │ + getRetryDelay()    │
                                 └──────────────────────┘
```

---

## Performance Characteristics

### Time Complexity
- `scheduleJob()`: O(log N) - PriorityQueue insertion
- `getJob()`: O(1) - HashMap lookup
- `executeJob()`: O(1) - Thread pool submission
- `dispatch()`: O(log N) - PriorityQueue poll

### Space Complexity
- Job storage: O(N) where N = total jobs
- Execution history: O(N × M) where M = avg executions per job
- Job queue: O(P) where P = pending jobs

---

## Sequence Diagrams

### 1. End-to-End Job Scheduling and Execution Flow

```
Client          JobScheduler    JobRepository    JobDispatcher    PriorityQueue    JobExecutor    ThreadPool
  |                  |                |                 |                |               |              |
  |--createJob()---->|                |                 |                |               |              |
  |                  |                |                 |                |               |              |
  |--scheduleJob()-->|                |                 |                |               |              |
  |                  |                |                 |                |               |              |
  |                  |--save(job)---->|                 |                |               |              |
  |                  |<---------------|                 |                |               |              |
  |                  |                |                 |                |               |              |
  |                  |--dispatch()----+---------------->|                |               |              |
  |                  |                |                 |                |               |              |
  |                  |                |                 |--offer()------>|               |              |
  |                  |                |                 |                | [job queued]  |              |
  |<--jobId----------|                |                 |                |               |              |
  |                  |                |                 |                |               |              |
  |                  |                |                 |[dispatcher loop]               |              |
  |                  |                |                 |                |               |              |
  |                  |                |                 |--poll()------->|               |              |
  |                  |                |                 |<--job----------|               |              |
  |                  |                |                 |                |               |              |
  |                  |                |                 |--check execution time          |              |
  |                  |                |                 |  (is now >= scheduled time?)   |              |
  |                  |                |                 |                |               |              |
  |                  |                |                 |--submit(job)---+-------------->|              |
  |                  |                |                 |                |               |              |
  |                  |                |                 |                |               |--execute()-->|
  |                  |                |                 |                |               |              |
  |                  |                |                 |                |               |<--thread-----|
  |                  |                |                 |                |               |              |
  |                  |                |                 |                |               |--setStatus(RUNNING)
  |                  |                |                 |                |               |              |
  |                  |                |                 |                |               |--job.execute()
  |                  |                |                 |                |               |              |
  |                  |                |                 |                |               |--setStatus(COMPLETED)
  |                  |                |                 |                |               |              |
  |                  |                |                 |                |               |--update()-->|
  |                  |                |<----------------|----------------|---------------|              |
  |                  |                |                 |                |               |              |
```

### 2. Job Scheduling with Delayed Execution

```
Client          JobScheduler    JobRepository    JobDispatcher    PriorityQueue    Time
  |                  |                |                 |                |            |
  |--scheduleDelayedJob(job, 5000ms)->|                 |                |            |
  |                  |                |                 |                |            |
  |                  |--calculate scheduledTime          |                |            |
  |                  |  (now + 5000ms)|                 |                |            |
  |                  |                |                 |                |            |
  |                  |--save(job)---->|                 |                |            |
  |                  |<---------------|                 |                |            |
  |                  |                |                 |                |            |
  |                  |--dispatch()----+---------------->|                |            |
  |                  |                |                 |                |            |
  |                  |                |                 |--offer()------>|            |
  |<--jobId----------|                |                 |                |            |
  |                  |                |                 |                | [sorted by |
  |                  |                |                 |                |  exec time]|
  |                  |                |                 |                |            |
  |                  |                |                 |[dispatcher loop]            |
  |                  |                |                 |                |            |
  |                  |                |                 |--poll()------->|            |
  |                  |                |                 |<--job----------|            |
  |                  |                |                 |                |            |
  |                  |                |                 |--check time--->|            |
  |                  |                |                 |  (now < scheduled?)         |
  |                  |                |                 |                |            |
  |                  |                |                 |--sleep(1s)-----|----------->|
  |                  |                |                 |                |            |
  |                  |                |                 |--re-queue----->|            |
  |                  |                |                 |                |            |
  |                  |                |                 |                |  [5s later]|
  |                  |                |                 |                |<-----------|
  |                  |                |                 |                |            |
  |                  |                |                 |--poll()------->|            |
  |                  |                |                 |<--job----------|            |
  |                  |                |                 |                |            |
  |                  |                |                 |--check time--->|            |
  |                  |                |                 |  (now >= scheduled)         |
  |                  |                |                 |                |            |
  |                  |                |                 |--submit to executor         |
  |                  |                |                 |                |            |
```

### 3. Job Execution with Retry Flow

```
JobExecutor    Job         RetryPolicy    JobRepository    JobDispatcher    ExecutionContext
    |           |               |               |                |                |
    |--executeJob(job)--------->|               |                |                |
    |           |               |               |                |                |
    |--create ExecutionContext-|---------------|----------------|--------------->|
    |           |               |               |                |                |
    |--setStatus(RUNNING)------>|               |                |                |
    |           |               |               |                |                |
    |--update(job)--------------|-------------->|                |                |
    |           |               |               |                |                |
    |--job.execute()----------->|               |                |                |
    |           |               |               |                |                |
    |           |--[EXCEPTION]->|               |                |                |
    |           |               |               |                |                |
    |--handleFailure()--------->|               |                |                |
    |           |               |               |                |                |
    |--canRetry()?------------->|               |                |                |
    |           |               |               |                |                |
    |           |--YES--------->|               |                |                |
    |           |               |               |                |                |
    |--incrementRetryCount()-->|               |                |                |
    |           |               |               |                |                |
    |--setStatus(RETRYING)---->|               |                |                |
    |           |               |               |                |                |
    |--getRetryDelay(attemptNum)|------------->|                |                |
    |           |<--------------|---------------|                |                |
    |           |  [2000ms for attempt 1]       |                |                |
    |           |               |               |                |                |
    |--scheduleRetry(job, 2000ms)               |                |                |
    |           |               |               |                |                |
    |           |  [wait 2000ms]|               |                |                |
    |           |               |               |                |                |
    |--submit(job again)--------|---------------|--------------->|                |
    |           |               |               |                |                |
    |--job.execute()----------->|               |                |                |
    |           |               |               |                |                |
    |           |--SUCCESS----->|               |                |                |
    |           |               |               |                |                |
    |--setStatus(COMPLETED)--->|               |                |                |
    |           |               |               |                |                |
    |--update(job)--------------|-------------->|                |                |
    |           |               |               |                |                |
    |--markCompleted()----------|---------------|----------------|--------------->|
    |           |               |               |                |                |
```

### 4. Priority-Based Job Dispatching

```
Client          JobScheduler    PriorityQueue    JobDispatcher    JobExecutor
  |                  |                |                |                |
  |--schedule Job A (LOW priority)--->|                |                |
  |                  |--offer()------>| [A]            |                |
  |                  |                |                |                |
  |--schedule Job B (HIGH priority)-->|                |                |
  |                  |--offer()------>| [B, A]         |                |
  |                  |                | (sorted)       |                |
  |                  |                |                |                |
  |--schedule Job C (NORMAL priority)>|                |                |
  |                  |--offer()------>| [B, C, A]      |                |
  |                  |                | (priority order)|               |
  |                  |                |                |                |
  |                  |                |                |[dispatch loop] |
  |                  |                |                |                |
  |                  |                |<--poll()-------|                |
  |                  |                |--Job B-------->|                |
  |                  |                |                |                |
  |                  |                |                |--submit(B)---->|
  |                  |                |                |                |
  |                  |                |<--poll()-------|                |
  |                  |                |--Job C-------->|                |
  |                  |                |                |                |
  |                  |                |                |--submit(C)---->|
  |                  |                |                |                |
  |                  |                |<--poll()-------|                |
  |                  |                |--Job A-------->|                |
  |                  |                |                |                |
  |                  |                |                |--submit(A)---->|
  |                  |                |                |                |
```

### 5. Job Cancellation Flow

```
Client          JobScheduler    JobRepository    Job
  |                  |                |            |
  |--getJob(jobId)-->|                |            |
  |                  |                |            |
  |                  |--findById()-->|            |
  |                  |<--job---------|            |
  |                  |                |            |
  |<--job------------|                |            |
  |                  |                |            |
  |--cancelJob(jobId)>|               |            |
  |                  |                |            |
  |                  |--findById()-->|            |
  |                  |<--job---------|            |
  |                  |                |            |
  |                  |--validate status            |
  |                  |  (cannot cancel RUNNING)    |
  |                  |                |            |
  |                  |--setStatus(CANCELLED)------>|
  |                  |                |            |
  |                  |--update(job)-->|            |
  |                  |<---------------|            |
  |                  |                |            |
  |<--true-----------|                |            |
  |                  |                |            |
```

### 6. Recurring Job Flow

```
Client          JobScheduler    CronExpression    JobDispatcher    PriorityQueue
  |                  |                 |                |                |
  |--scheduleRecurringJob(job, "0 0 * * *")----------->|                |
  |                  |                 |                |                |
  |                  |--save(job)----->|                |                |
  |                  |                 |                |                |
  |                  |--create ScheduledJob(cron)------>|                |
  |                  |                 |                |                |
  |                  |                 |--calculateNextExecution()       |
  |                  |                 |  (tomorrow midnight)            |
  |                  |                 |                |                |
  |                  |                 |                |--offer()------>|
  |<--jobId----------|                 |                |                |
  |                  |                 |                |                |
  |                  |                 |                |[dispatcher loop]
  |                  |                 |                |                |
  |                  |                 |                |--poll()------->|
  |                  |                 |                |<--job----------|
  |                  |                 |                |                |
  |                  |                 |                |--check time--->|
  |                  |                 |                |  (tomorrow?)   |
  |                  |                 |                |                |
  |                  |                 |                |[next day]      |
  |                  |                 |                |                |
  |                  |                 |                |--submit to executor
  |                  |                 |                |                |
  |                  |                 |                |--getNextOccurrence()
  |                  |                 |<---------------|                |
  |                  |                 |                |                |
  |                  |                 |--calculateNextExecution()       |
  |                  |                 |  (day after tomorrow)           |
  |                  |                 |                |                |
  |                  |                 |                |--offer()------>|
  |                  |                 |                | [reschedule]   |
  |                  |                 |                |                |
```

### 7. Complete System Flow with Persistence

```
Client      JobScheduler    Repository    Dispatcher    Executor    Database
  |              |               |             |            |            |
  |--schedule()->|               |             |            |            |
  |              |               |             |            |            |
  |              |--save()------>|-------------|------------|---------->|
  |              |<--------------|-------------|------------|-----------|
  |              |               |             |            |            |
  |              |--dispatch()-->|             |            |            |
  |              |               |--offer()    |            |            |
  |              |               |             |            |            |
  |<--jobId------|               |             |            |            |
  |              |               |             |            |            |
  |              |               |[poll & check]            |            |
  |              |               |             |            |            |
  |              |               |             |--submit()->|            |
  |              |               |             |            |            |
  |              |               |             |            |--execute() |
  |              |               |             |            |            |
  |              |               |             |            |--update()>|
  |              |               |<------------|------------|-----------|
  |              |               |   (status: RUNNING)      |            |
  |              |               |             |            |            |
  |              |               |             |            |[job runs]  |
  |              |               |             |            |            |
  |              |               |             |            |--update()>|
  |              |               |<------------|------------|-----------|
  |              |               |   (status: COMPLETED)    |            |
  |              |               |             |            |            |
  |              |               |             |            |--saveExecContext()
  |              |               |<------------|------------|---------->|
  |              |               |             |            |            |
  |              |               |             |            |            |
  |  [system restart]            |             |            |            |
  |              |               |             |            |            |
  |              |--start()----->|             |            |            |
  |              |               |             |            |            |
  |              |               |--findByStatus(SCHEDULED)>|---------->|
  |              |               |<-------------------------|-----------|
  |              |               | [pending jobs]           |            |
  |              |               |             |            |            |
  |              |               |--for each job            |            |
  |              |               |  dispatch()>|            |            |
  |              |               |             |--offer()   |            |
  |              |               |             |            |            |
```

---

## Summary

This Job Scheduler System demonstrates:

1. **Complete Lifecycle**: Job creation → persistence → dispatching → execution → completion
2. **Priority Scheduling**: Priority-based job ordering with time-based execution
3. **Fault Tolerance**: Retry mechanism with exponential backoff
4. **Persistence**: Repository pattern for job storage and history
5. **Concurrency**: Thread-safe operations with thread pool execution
6. **Extensibility**: Abstract job class for custom implementations
7. **Monitoring**: Comprehensive stats and execution tracking
8. **Scalability**: Configurable thread pools and queue management
9. **Production Features**: Dead letter queue, job chaining, dependency management

The design is suitable for Staff Engineer level interviews, covering distributed systems concepts, concurrency patterns, and production-ready architecture.
