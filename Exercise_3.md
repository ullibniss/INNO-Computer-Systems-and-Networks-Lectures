# CSN. Exercise 3

## Done by Fedorov Alexey and Joel Okore [M24-SNE-01]

---

```
Scenario: You are tasked with processing a stream of sensor data where each sensor reading is a string inthe format:

sensor_id,timestamp,value
 Example: 001, 2024-09-01 12:00:00, 25.3

The goal is to:
• Filter sensor readings to only include those with values above 20.
• Transform the readings into a JSON format.
• Store the results in a file named processed_readings.json.

Part 1: Pipes and Filters:
• Implement this pipeline using the Pipes and Filters pattern.
• Each stage of the pipeline (filter, transformation, storage) should be a separate module (function or class), connected by pipes (streams or queues).
• Ensure each filter is stateless and processes data as a stream.

Part 2: Producer-Consumer:
- Implement the same task using the **Producer-Consumer** model.
- Have one producer that reads sensor data and places it in a shared buffer.
- Multiple consumers will:
- Consumer 1: Filter the data.
- Consumer 2: Transform the filtered data.
- Consumer 3: Write the transformed data to a file.
- Use a queue to handle communication between the producer and consumers.

Comparison: After implementing both systems, answer the following questions:
1. How do Pipes and Filters differ from Producer-Consumer in terms of:
1. Modularity (how easy it is to swap or modify a step in the pipeline)?
2. Scalability (adding more filters or consumers)?
3. Data handling (synchronous vs. asynchronous processing)?
2. Which approach seems easier to manage as the complexity of the pipeline grows?
3. In what scenarios would you choose one approach over the other?
Hint:
• Use a queue data structure to simulate pipes in the Pipes and Filters pattern.
• For the Producer-Consumer model, implement a simple shared buffer with thread synchronization (like locks or semaphores) to manage concurrent access.
• Focus on the modularity and how each approach handles the stream of sensor data differently.
```

## Part 1: Pipes and Filters

In this task, We will implement the "Pipes and Filters" pattern for sensor data processing. We prefer to use the Go programming language. Let's start.

First, We defined a structure for sensor data.

```golang
type SensorData struct {
	Sensor_id int     `json:"sensor_id"`
	Date      string  `json:"date"`
	Value     float64 `json:"value"`
}
```

**SensorData** - a structure where all the fields required for storing sensor data are present.

A queue data structure is used to simulate pipes. The implementation is:

```golang
type Pipe chan []byte

func EncodeToBytes(p interface{}) []byte {
	buf := bytes.Buffer{}
	enc := gob.NewEncoder(&buf)
	err := enc.Encode(p)
	if err != nil {
		log.Fatal(err)
	}
	return buf.Bytes()
}

func DecodeToSensorData(b []byte) SensorData {
	s := SensorData{}
	dec := gob.NewDecoder(bytes.NewReader(b))
	err := dec.Decode(&s)
	if err != nil {
		log.Fatal(err)
	}
	return s
}
```

**Pipe** - a `chan` type structure that imitates a queue of `[]byte` data. We decided to use `[]byte` to represent a binary stream.  
**EncodeToBytes**, **DecodeToSensorData** - methods for marshaling and unmarshaling `SensorData` to and from bytes.

We will receive a **string** as input. Therefore, we need to convert this data into the `SensorData` structure. The following method is implemented for this purpose:

```golang
func ParseRawData(rawData string) []SensorData {
	sensorsData := make([]SensorData, 0)
	rows := strings.Split(rawData, "\n")
	for _, row := range rows {
		if row == "" {
			continue
		}
		rowSplit := strings.Split(row, ", ")
		sensorId, _ := strconv.Atoi(rowSplit[0])
		value, _ := strconv.ParseFloat(rowSplit[2], 64)
		sensorsData = append(sensorsData, SensorData{
			sensorId,
			rowSplit[1],
			value,
		})
	}
	return sensorsData
}
```

It parses the string and returns an array of sensor data.

Let's implement our flow functions:

### Filter

```golang
func Filter(in Pipe, out Pipe) {
	for data := range in {
		sensorData := DecodeToSensorData(data)
		if sensorData.Value > 20 {
			out <- EncodeToBytes(sensorData)
		}
	}
	close(out)
}
```

**Filter** - filter sensor readings to only include those with values above 20.

### Transform

```golang
func Transform(in Pipe, out Pipe) {
	for data := range in {
		sensorData := DecodeToSensorData(data)
		jsonSensorData, _ := json.Marshal(sensorData)
		out <- jsonSensorData
	}
	close(out)
}
```

**Transform** - encodes the readings into a JSON format.

### Store

```golang
func Store(in Pipe) {
	fo, err := os.OpenFile("processed_readings.json", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		panic(err)
	}
	defer func() {
		if err := fo.Close(); err != nil {
			panic(err)
		}
	}()

	for data := range in {
		if _, err := fo.Write(append(data, '\n')); err != nil {
			panic(err)
		}
		fmt.Println("Data saved to file processed_readings.json")
	}
}
```

**Store** -stores the results in a file named processed_readings.json.

All functions for building the pipeline are implemented. Let's assemble it in the `main` function:

```golang
func main() {
	var rawData string
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		line := scanner.Text()
		if line == "" {
			break
		}
		rawData += line + "\n"
	}

	pipe1 := make(Pipe)
	pipe2 := make(Pipe)
	pipe3 := make(Pipe)

	sensorsData := ParseRawData(rawData)

	var wg sync.WaitGroup
	wg.Add(3)

	go func() {
		defer wg.Done()
		Filter(pipe1, pipe2)
	}()

	go func() {
		defer wg.Done()
		Transform(pipe2, pipe3)

	}()

	go func() {
		defer wg.Done()
		Store(pipe3)
	}()

	for _, data := range sensorsData {
		pipe1 <- EncodeToBytes(data)
	}
	close(pipe1)

	wg.Wait()
}
```

1. **Scanner** reads the string from input.
2. Three pipelines are created: `pipe1`, `pipe2`, `pipe3`.
3. The raw string is parsed into `[]SensorData`.
4. A **WaitGroup** is created to wait for asynchronous processes.
5. Three **goroutines** are created with filter functions, using the pipes as parameters.
6. Data is sent to `pipe1` to start the data processing.

## Part 2: Producer-Consumer

For producer-consumer we implemented shared buffer:

```golang
type SharedBuffer struct {
	buffer   []string
	capacity int
	mu       sync.Mutex
	notEmpty *sync.Cond
	notFull  *sync.Cond
	done     bool
}

func NewSharedBuffer(capacity int) *SharedBuffer {
	sb := &SharedBuffer{
		buffer:   make([]string, 0, capacity),
		capacity: capacity,
	}
	sb.notEmpty = sync.NewCond(&sb.mu)
	sb.notFull = sync.NewCond(&sb.mu)
	return sb
}

func (sb *SharedBuffer) Add(item string) {
	sb.mu.Lock()
	defer sb.mu.Unlock()

	for len(sb.buffer) == sb.capacity {
		sb.notFull.Wait()
	}

	sb.buffer = append(sb.buffer, item)

	sb.notEmpty.Signal()
}

func (sb *SharedBuffer) Remove() (string, bool) {
	sb.mu.Lock()
	defer sb.mu.Unlock()

	for len(sb.buffer) == 0 && !sb.done {
		sb.notEmpty.Wait()
	}

	if len(sb.buffer) == 0 && sb.done {
		return "", false
	}

	item := sb.buffer[0]
	sb.buffer = sb.buffer[1:]

	sb.notFull.Signal()

	return item, true
}

func (sb *SharedBuffer) SetDone() {
	sb.mu.Lock()
	defer sb.mu.Unlock()
	sb.done = true
	sb.notEmpty.Broadcast()
}
```


It supports synchronization and concurrent access to data.

Let's implement the producer. It is a simple function:

```golang
func producer(data []string, sb *SharedBuffer, wg *sync.WaitGroup) {
	defer wg.Done()
	for _, entry := range data {
		sb.Add(entry)
	}
	sb.SetDone()
}
```

Function takes data and buffer, then send data to buffer.

Three main consumers are:
- **consumerFilter** - filter sensor readings to only include those with values above 20.
- **consumerTranform** - transforms the readings into a JSON format.
- **consumerStore** - stores the results in a file named processed_readings.json.

The implementation is:

```golang
func consumerFilter(sb *SharedBuffer, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		entry, more := sb.Remove()
		if !more {
			return
		}

		record := strings.Split(entry, ",")
		if len(record) != 3 {
			continue
		}

		sensorID := record[0]
		timestamp, _ := time.Parse("2006-01-02 15:04:05", strings.TrimSpace(record[1]))
		value, _ := strconv.ParseFloat(strings.TrimSpace(record[2]), 64)

		if value > 20 {
			filteredEntry := fmt.Sprintf("%s,%s,%.2f", sensorID, timestamp.Format("2006-01-02 15:04:05"), value)
			sb.Add(filteredEntry)
		}
	}
}

func consumerTransform(sb *SharedBuffer, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		entry, more := sb.Remove()
		if !more {
			return
		}

		record := strings.Split(entry, ",")
		if len(record) != 3 {
			continue
		}

		sensorID := record[0]
		timestamp, _ := time.Parse("2006-01-02 15:04:05", strings.TrimSpace(record[1]))
		value, _ := strconv.ParseFloat(strings.TrimSpace(record[2]), 64)

		reading := SensorData{
			SensorID: sensorID,
			Date:     timestamp,
			Value:    value,
		}
		jsonData, err := json.Marshal(reading)
		if err == nil {
			sb.Add(string(jsonData))
		}
	}
}

func consumerStorage(sb *SharedBuffer, wg *sync.WaitGroup) {
	defer wg.Done()
	file, err := os.Create("processed_readings.json")
	if err != nil {
		fmt.Println("Error creating file:", err)
		return
	}
	defer file.Close()

	for {
		entry, more := sb.Remove()
		if !more {
			return
		}

		file.WriteString(entry + "\n")
	}

	fmt.Println("Data successfully written to processed_readings.json")
}
```

Now we have all the parts to build the producer-consumer flow:

```
func main() {

	var data []string
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		line := scanner.Text()
		if line == "" {
			break
		}
		data = append(data, line)
	}
	
	sharedBuffer := NewSharedBuffer(2)

	var wg sync.WaitGroup
	
	wg.Add(1)
	go producer(data, sharedBuffer, &wg)
	
	wg.Add(3)
	go consumerFilter(sharedBuffer, &wg)
	go consumerTransform(sharedBuffer, &wg)
	go consumerStorage(sharedBuffer, &wg)
	
	wg.Wait()
}
```

1. **Scanner** reads the string from input.
2. A **shared buffer** is created.
3. A **WaitGroup** is created to wait for asynchronous processes.
4. One **goroutine** is created with the producer function, which takes data and the shared buffer as parameters.
5. Three **goroutines** are created with consumer functions, each taking the shared buffer as a parameter.
6. Data processing starts.

## Comparition

### 1. Comparison of **Pipe-Filters** and **Producer-Consumer** Patterns

#### 1.1 **Modularity**
- **Pipe-Filters**: 
   - *High Modularity*: The pipe-filter pattern is inherently modular. Each filter (processing step) is self-contained and follows the same interface, making it easy to swap or modify filters without impacting other parts of the pipeline. Filters are designed to be loosely coupled, meaning each filter doesn't need to know the internal workings of the others.
   - *Ease of modification*: This modularity makes it easy to insert, remove, or replace filters without disrupting the overall structure of the pipeline.

- **Producer-Consumer**:
   - *Moderate Modularity*: Producer-Consumer typically involves a more direct relationship between components (producers and consumers). While it can be made modular, the pattern often focuses on one producer feeding into one or more consumers. Changing the structure (e.g., adding a consumer or modifying the producer) may require adjustments to how the data is shared and coordinated.
   - *More interdependencies*: Swapping or modifying a step is possible but may involve more configuration or synchronization code compared to Pipe-Filters.

#### 1.2 **Scalability**
- **Pipe-Filters**:
   - *Scales well in terms of adding more filters*: You can easily add more filters into the pipeline by simply connecting them through a data stream. Since filters are decoupled and usually communicate through standard interfaces, expanding the system doesn't require reconfiguring other parts of the pipeline.
   - *Challenge in parallelism*: If each filter needs to be processed in sequence, it may limit scalability when trying to parallelize tasks.

- **Producer-Consumer**:
   - *Scales well with more consumers*: The Producer-Consumer pattern is built around the idea of multiple consumers processing data produced by one or more producers. Adding more consumers can help distribute the workload, making it easier to handle increased data volumes. Similarly, you can scale by adding more producers as well.
   - *Parallelism-friendly*: Producer-Consumer systems are naturally suited to parallel processing because producers and consumers often work asynchronously. Adding more consumers can balance the load and allow for better utilization of resources in a highly parallelized system.

#### 1.3 **Data Handling (Synchronous vs. Asynchronous Processing)**
- **Pipe-Filters**:
   - *Often synchronous but can be asynchronous*: In its basic form, Pipe-Filters is often implemented synchronously, where one filter completes its work and passes the output to the next filter. However, it can be adapted to be asynchronous by adding buffers or queues between filters, enabling filters to work at their own pace without blocking others.
   - *Data passes step-by-step*: Data flows in a defined sequence through the filters, and each step may wait for the previous one to finish.

- **Producer-Consumer**:
   - *Typically asynchronous*: The Producer-Consumer pattern is usually implemented asynchronously, where producers generate data independently from the rate at which consumers process it. This is often achieved using queues that decouple the producer from the consumer, allowing both to work at their own pace without waiting for each other.
   - *More natural decoupling*: The asynchronous nature helps manage bursts in data production, as consumers can catch up when producers are faster, or producers can continue working even if consumers are temporarily busy.

### 2. **Ease of Management as Complexity Grows**

- **Pipe-Filters**:
   - *Moderate to High Management Complexity*: As the pipeline grows, managing the flow of data through a sequence of filters can become more complex. You need to ensure that the filters are connected in the right order, that the interfaces between filters are compatible, and that any buffering or error handling between steps is well-managed.
   - *Readable but potentially deep chains*: With more filters, you end up with a longer pipeline, which may become harder to troubleshoot or refactor as complexity increases.

- **Producer-Consumer**:
   - *Easier to Manage with Parallel Processing*: The Producer-Consumer pattern is typically easier to scale and manage in terms of workload distribution and parallelism, especially when you have multiple producers and consumers. With queues between producers and consumers, you can adjust each component’s capacity independently, making it easier to fine-tune performance.
   - *Queue management*: However, you need to manage queuing effectively and ensure that consumer threads are balanced. Too many consumers or producers can create contention or underutilization issues, but the decoupling aspect makes the system easier to manage in terms of handling bursts of data and concurrency.


### 3. **Scenarios for Choosing One Approach Over the Other**

#### **Choose Pipe-Filters When**:
- **Linear Data Flow**: The data processing tasks are highly sequential and independent, where each step performs a distinct operation and passes results to the next.
- **Modularity is Crucial**: You want a highly modular and easy-to-understand system where adding or removing filters does not impact the rest of the system.
- **Smaller Systems**: The system is relatively small or has well-defined processing steps that are not performance-critical.
- **Batch Processing**: You’re dealing with a pipeline that needs to process data in well-structured batches or streams where each step is tightly coupled with the next.

#### **Choose Producer-Consumer When**:
- **Concurrency and Parallelism**: There is a need for parallel processing, where multiple consumers need to process the data concurrently, and you want to scale the system horizontally.
- **Decoupled Components**: You want to decouple data production from data consumption so that each component can work independently at its own pace (asynchronous processing).
- **Handling Variable Workloads**: The system needs to handle bursts in data production or consumption, with queues helping manage the load.
- **Distributed Systems**: In scenarios where different machines or microservices handle different roles (e.g., cloud-based systems), Producer-Consumer with asynchronous messaging is highly suited.


### Summary
- **Pipe-Filters** excels in modularity and managing linear, sequential data flows with clear steps, but it may become complex as the system grows, especially when parallelism is needed.
- **Producer-Consumer** shines in handling parallel workloads, asynchronous data handling, and distributed systems but may require more careful management of queues and consumer/producer balance.

