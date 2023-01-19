# SimpleKafkaConsumer
Simple Kafka Consumer to read messages from beginning

C# .NET 6.0 Console

```
using Confluent.Kafka;
var NUM_PARTITIONS = 31;

Console.WriteLine("Starting consumer....");

var conf = new ConsumerConfig
{
	GroupId = "SomeConsumerGroup",
	BootstrapServers = "someServer:9093",
	SecurityProtocol = SecurityProtocol.SaslSsl,
	// Note: The AutoOffsetReset property determines the start offset in the event
	// there are not yet any committed offsets for the consumer group for the
	// topic/partitions of interest. By default, offsets are committed
	// automatically, so in this example, consumption will only start from the
	// earliest message in the topic 'my-topic' the first time you run the program.,
	SaslMechanism = SaslMechanism.Plain,
	SaslUsername = "$ConnectionString",
	SaslPassword = "Endpoint=sb://someServer/;SharedAccessKeyName=someKeyName;SharedAccessKey=someKey;EntityPath=somePath",
	AutoOffsetReset = AutoOffsetReset.Latest,
	//BrokerVersionFallback = "1.0.0",
	SocketKeepaliveEnable = false,
	//EnableAutoCommit = false,
	//EnableAutoOffsetStore = false,
};


for (var i = 0; i < NUM_PARTITIONS; i++)
{
	new Thread(() =>
	{
		Thread.CurrentThread.IsBackground = true;
		InitConsumer(i);
	}).Start();
}

Console.ReadLine();


void InitConsumer(int partition)
{

	using (var c = new ConsumerBuilder<Ignore, string>(conf)
		/*.SetPartitionsAssignedHandler((c, partitions) =>
		{
			var offsets = partitions.Select(tp => new TopicPartitionOffset(tp, Offset.Beginning));
			return offsets;
		})*/
		.Build())
	{
		c.Subscribe("someTopic");

		CancellationTokenSource cts = new CancellationTokenSource();
		Console.CancelKeyPress += (_, e) =>
		{
			e.Cancel = true; // prevent the process from terminating.
			cts.Cancel();
		};

		TopicPartitionOffset tps = new TopicPartitionOffset(new TopicPartition("someTopic", partition), Offset.Beginning);
		c.Assign(tps);

		try
		{
			while (true)
			{
				try
				{
					var cr = c.Consume(cts.Token);					
					Console.WriteLine($"Consumed message: {cr.Value} '{cr.TopicPartitionOffset}'.");
				}
				catch (ConsumeException e)
				{
					Console.WriteLine($"Error occured: {e.Error.Reason}");
				}
			}
		}
		catch (OperationCanceledException)
		{
			// Ensure the consumer leaves the group cleanly and final offsets are committed.
			c.Close();
		}
	}
}
```
