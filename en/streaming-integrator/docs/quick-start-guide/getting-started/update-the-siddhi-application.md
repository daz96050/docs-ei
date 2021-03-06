# Step 6: Update the Siddhi Application

A Siddhi application can be easily extended to consume messages from more sources, to carry out more processing activities for data or to publish data to more destinations. For this example, consider a scenario where you also need to filter out the production data of eclairs and publish it to a Kafka topic so that applications that cannot read streaming data can have access to it. This involves extending the `SweetFactoryApp` Siddhi application to include Kafka in the streaming flow. To do this, follow the steps below:

1. Open the `<SI_HOME>/wso2/server/deployment/siddhi-files/SweetFactoryApp` Siddhi application in a text editor of your choice.

    !!! tip
        Alternatively, you can open this file in Streaming Integrator Tooling and deploy it again after completing the changes and saving it.

2. Define another stream to which you can direct the filtered events you need to publish in a the Kafka topic.

    `define stream FilterStream (name string,amount double);`
    
3. To publish the events filtered into the `PublishFilteredDataStream` stream, connect a source of the `kafa` type to it as shown below.

    ```
    @sink(type = 'kafka', bootstrap.servers = "localhost:9092", topic = "eclair_production", is.binary.message = "false", partition.no = "0",
             @map(type = 'json'))
    define stream PublishFilteredDataStream (name string,amount double);
    ```
   
   The above sink annotation publishes all the events received into the `PublishFilteredDataStream` stream into a topic named `eclair_production` in `json` format.
   
4. Let's create another stream to read from the `/Users/foo/productioninserts.csv` file to which you have been publishing data.

    !!! tip
        Alternatively, you can write the query to read from one of the existing streams. However, in this example, let's create a new stream to understand how WSO2 Streaming Integrator reads data from files.
        
    ```
    @source(type='file', mode='LINE',
       file.uri='file:/Users/foo/productions.csv',
       tailing='true',
       @map(type='csv'))
    define stream FilterStream (name string,amount double);
    ```
   
    Here, you are configuring the file to be read line by line in the tailing mode. Therefore, any new row added to the file is captured as an event in the `FilterStream` stream as and when it is added.


5. Now let's add the query to filter the required information and publish it.

    ```
    from FilterStream [name=='Eclairs']
    select * 
    group by name 
    insert  into PublishFilteredDataStream;
    ```
   
    In the `from` clause, `[name=='Eclairs']` filters all production runs where the name of the sweet produced is `Eclairs`. Then all the filtered events are inserted into the `PublishFilteredDataStream` stream so that they can be published in the `eclair_production` Kafka topic.
    
6. Save your changes.

The completed Siddhi application looks as follows:

```
@App:name('SweetFactoryApp')

@App:statistics(reporter = 'prometheus')


@source(type='cdc',url = "jdbc:mysql://localhost:3306/production",username = "wso2si",password = "wso2",table.name = "SweetProductionTable",operation = "insert",
	@map(type='keyvalue'))
define stream InsertSweetProductionStream (name string,amount double);

@source(type='file', mode='LINE',
   file.uri='file:/Users/foo/productions.csv',
   tailing='true',
   @map(type='csv'))
define stream FilterStream (name string,amount double);

@sink(type='file',file.uri = "/Users/foo/productioninserts.csv",
	@map(type='text'))
define stream ProductionUpdatesStream (name string,amount double);

@sink(type = 'kafka', bootstrap.servers = "localhost:9092", topic = "eclair_production", is.binary.message = "false", partition.no = "0",
         @map(type = 'json'))
define stream PublishFilteredDataStream (name string,amount double);

@info(name='query1')
from InsertSweetProductionStream 
select str:upper(name) as name, amount 
group by name 
insert  into ProductionUpdatesStream;

from FilterStream [name=='Eclairs']
select * 
group by name 
insert  into PublishFilteredDataStream;
```