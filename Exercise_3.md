# CSN. Exercise 3

## Done by Fedorov Alexey

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

In this task i will implement "Pipes and Filters" pattern for sensor data processing. I prefer to use Golang language. Let's start.

First, I defined structure for Sensor data.

```golang
type SensorData struct {
	Sensor_id int     `json:"sensor_id"`
	Date      string  `json:"date"`
	Value     float64 `json:"value"`
}
```

**SensorData** - structure where all field needed for sendor data store present.

There is queue data structure to sumulate pipes in Hint. Implementation is:

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

**Pipe** - chan type structure, that imitates queue of []byte data. I decided to use []byte to have binary stream.
**EncodeToBytes**, **DecodeToSensprData** - method for marshaling and unmarshling SensorData to bytes.

We will have **string** on input. Because of this we need to convert this data to SensorData structure. The following method implemented for this purpose:

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

It parses string and returns array of sensor data.

Let's implement our flow fucntions:

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

All fucntions for pipeline build are implemented. Let's build it in main function:
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

1. Scanner reads string from input.
2. Three pipelines are created: `pipe1`, `pipe2`, `pipe3`.
3. Raw string parsed to []SensorData.
4. Create Wait Group for async processes waiting.
5. Created 3 gorutines with filter fucntions. They have pipes are parameters.
6. Data sent to pipe1, to start data processing.

## Part 2: Producer-Consumer

