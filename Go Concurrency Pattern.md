#snippet

# Go: Concurrency Patterns

### Runner

Runner模式启动goroutine执行一系列tasks，等待任务完成或者在timeout或者系统中断时停止。
```go
// Runner runs a set of tasks within a given timeout and can be shut down on an operating system interrupt.
type Runner struct {
	// interrupt channel reports a signal from the operating system.
	interrupt chan os.Signal
  
	// complete channel reports that processing is done.
	complete chan error
  
	// timeout reports that time has run out.
	timeout <-chan time.Time
  
	// tasks holds a set of functions that are executed synchronously in index order.
	tasks []func(int)
}

// ErrTimeout is returned when a value is received on the timeout.
var ErrTimeout = errors.New("received timeout")

// ErrInterrupt is returned when an event from the OS is received.
var ErrInterrupt = errors.New("received interrupt")

// New returns a new ready-to-use Runner.
func New(d time.Duration) *Runner {
	return &Runner{
		interrupt: make(chan os.Signal, 1),
		complete:  make(chan error),
		timeout:   time.After(d),
	}
}

// Add attaches tasks to the Runner. A task is a function that takes an int ID.
func (r *Runner) Add(tasks ...func(int)) {
	r.tasks = append(r.tasks, tasks...)
}

// Start runs all tasks and monitors channel events.
func (r *Runner) Start() error {
	// We want to receive all interrupt based signals.
	signal.Notify(r.interrupt, os.Interrupt)
  
	// Run the different tasks on a different goroutine.
	go func() {
		r.complete <- r.run()
	}()
  
	select {
	// Signaled when processing is done.
	case err := <-r.complete:
		return err
      
	// Signaled when we run out of time.
	case <-r.timeout:
		return ErrTimeout
	}
}

// run executes each registered task.
func (r *Runner) run() error {
	for id, task := range r.tasks {
		// Check for an interrupt signal from the OS.
		if r.gotInterrupt() {
			return ErrInterrupt
		}
      
		// Execute the registered task.
		task(id)
	}
	return nil
}

// gotInterrupt verifies if the interrupt signal has been issued.
func (r *Runner) gotInterrupt() bool {

	select {
	// Signaled when an interrupt event is sent.
	case <-r.interrupt:
		// Stop receiving any further signals.
		signal.Stop(r.interrupt)
		return true
      
	// Continue running as normal.
	default:
		return false
	}
}
```

### Pool

Pool模式管理资源池，供多个goroutine使用。池中资源不足时，通过factory创建新资源。Pool关闭时Close资源。

获取资源时，通过`r, ok := <-p.resources`判断池中是否有资源

关闭池和释放资源时，需要加锁，防止往closed channel中写入释放的资源，造成panic。
```go
// Pool manages a set of resources that can be shared safely by multiple goroutines. The resource being managed must implement the io.Closer interface.
type Pool struct {
	m         sync.Mutex
	resources chan io.Closer
	factory   func() (io.Closer, error)
	closed    bool
}

// ErrPoolClosed is returned when an Acquire returns on a closed pool.
var ErrPoolClosed = errors.New("pool has been closed")

// New creates a pool that manages resources. A pool requires a function that can allocate a new resource and the size of the pool.
func New(fn func() (io.Closer, error), size uint) (*Pool, error) {
	if size <= 0 {
		return nil, errors.New("size value too small")
	}

	return &Pool{
		factory:   fn,
		resources: make(chan io.Closer, size),
	}, nil

}

// Acquire retrieves a resource from the pool.
func (p *Pool) Acquire() (io.Closer, error) {
	select {
	// Check for a free resource.
	case r, ok := <-p.resources:
		log.Println("Acquire:", "Shared Resource")
		if !ok {
			return nil, ErrPoolClosed
		}
		return r, nil
	// Provide a new resource since there are none available.
	default:
		log.Println("Acquire:", "New Resource")
		return p.factory()
	}
}

// Release places a new resource onto the pool.
func (p *Pool) Release(r io.Closer) {
	// Secure this operation with the Close operation.
	p.m.Lock()
	defer p.m.Unlock()
	// If the pool is closed, discard the resource.
	if p.closed {
		r.Close()
		return
	}
	select {
	// Attempt to place the new resource on the queue.
	case p.resources <- r:
		log.Println("Release:", "In Queue")
	// If the queue is already at capacity we close the resource.
	default:
		log.Println("Release:", "Closing")
		r.Close()
	}
}

// Close will shutdown the pool and close all existing resources.
func (p *Pool) Close() {
	// Secure this operation with the Release operation.
	p.m.Lock()
	defer p.m.Unlock()

	// If the pool is already closed, don't do anything.
	if p.closed {
		return
	}
	// Set the pool as closed.
	p.closed = true

	// Close the channel before we drain the channel of its resources. If we don't do this, we will have a deadlock.
	close(p.resources)

	// Close the resources
	for r := range p.resources {
		r.Close()
	}
}
```


### Worker

Worker建立一个unbuffer的channel，启动多个goroutine worker消费（执行）发送到channel的任务。往channel添加任务时会阻塞，直到确保任务被某个goroutine获取执行。

channel关闭时，会等待所有worker结束。
```go
// Worker must be implemented by types that want to use the work pool.
type Worker interface {
	Task()
}

// Pool provides a pool of goroutines that can execute any Worker
// tasks that are submitted.
type Pool struct {
	work chan Worker
	wg   sync.WaitGroup
}

// New creates a new work pool.
func New(maxGoroutines int) *Pool {
	p := Pool{
		work: make(chan Worker),
	}
	p.wg.Add(maxGoroutines)
	for i := 0; i < maxGoroutines; i++ {
		go func() {
			for w := range p.work {
				w.Task()
			}
			p.wg.Done()
		}()
	}

	return &p
}

// Run submits work to the pool.
func (p *Pool) Run(w Worker) {
	p.work <- w
}

// Shutdown waits for all the goroutines to shutdown.
func (p *Pool) Shutdown() {
	close(p.work)
	p.wg.Wait()
}
```

