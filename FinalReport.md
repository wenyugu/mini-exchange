Mini Exchange Project Report
===
An Order Matching Engine, Gateway, and Tickerplant in Java SE 11

## Project Description

The project involves creating a mini exchange using Java which consists of a gateway for order submission using [FIX](https://en.wikipedia.org/wiki/Financial_Information_eXchange) protocol, one/multiple order matching engine(s), and a tickerplant that publishes market data in [IEX DEEP](https://iextrading.com/docs/IEX%20DEEP%20Specification.pdf) format using UDP multicast and websocket to stream data for frontend visualization as well as a web-based dashboard that can visualize and analyze market data in real-time. Clients can establish a direct TCP connection to the gateway, send MARKET, LIMIT, or CANCEL orders, and receive order execution reports. The matching engines receive the orders, check to see if they match any resting orders on the book, and then send execution report messages that contain information about the order status and any changes in price levels to the gateway and tickerplant using UDP multicast. The tickerplant maintains and updates a separate set of orderbooks upon receiving messages from matching engines and publishes market data using IEX DEEP format.   

Testing in the project is done by both unit tests and end-to-end tests that make multiple connections to the gateway, send batch orders, and then validate those that should match. We also test whether the appropriate execution report messages are received at the gateway and whether the correct market data updates are published from the tickerplant.

## Technology Overview

### Java SE 11
The [Java](http://www.java.com) programming language is one of the general-purpose programming languages that global programmers use. Java SE 11 is one of the long-term support (LTS) versions of Java. Java SE 11 is used in the project due to unresolved security issues in older versions and the compatibility of systems that the Mini Exchange project is consisted of. 

### Maven
[Apache Maven](https://maven.apache.org/), a tool developed by Apache Software Foundation, is a build automation tool that is especially used for Java-based programming projects. Maven is also compatible with C#, Ruby, Scala, and other languages. Maven pom file elaborates how software is built, building orders, dependencies, and required plug-ins. Maven targets to provide an easy build process, a uniform build system, quality project information, and encouragement for better development. Due to those advantages, Maven is chosen for building the Mini Exchange project.

### Log4j
[Apache Log4j](https://logging.apache.org/log4j/2.x/), a logging framework developed by Apache Software Foundation, is one of several Java logging frameworks.  

In the Mini Exchange project, Log4j 2 is used since it is highly configurable through external configuration files at runtime. Loggers can be easily configured to pretty format log messages for debugging and filter out logs according to its levels in production. For example, we might only care about logs involving ERRORS while filtering all WARNING logs.

### TCP Connection
The Transmission Control Protocol (TCP) is one of the main protocols of the Internet protocol suite. The TCP is unicast and runs on top of the Internet Protocol (IP). It provides reliability, ordering, error-detection, flow control, and invisibility to the user. Major internet applications that rely on TCP are World Wide Web, email, remote administration, and file transfer.  

TCP is connection-oriented, and a connection between the client and server must be established before the data can be sent. Before the client attempts the connection, the server has to be passively opened. Then, a client establishes the connection by using a three-way handshake: SYN, SYN/ACK, ACK. During active open, the client sends SYN, the server replies with SYN/ACK, and the client sends an ACK back to the server. Due to the acknowledgment and retransmission of the TCP connection, it is commonly used in sending orders for all exchange or broker interfaces.  

The Financial Information eXchange (FIX) protocol over TCP is used in the project for clients to submit orders. FIX is chosen because it is a global information and data protocol used to message trade information among financial institutions. The detailed description of FIX is discussed in the next section. 

### FIX Protocol
The [Financial Information eXchange (FIX) protocol](https://www.fixtrading.org/what-is-fix/) is an open electronic communications protocol that provides the fundamentals in facilitating the international real-time exchange of information related to securities transactions and markets over the past decade. The FIX protocol is comprised of a series of messaging specifications and is designed to standardize electronic communications in the financial services industry supporting multiple formats and types of communications between financial entities, including trade allocation, order submissions, order changes, execution reporting, and advertisements. Originally developed to support equities trading in the pre-trade and trade environment, it is now experiencing rapid expansion into the post-trade space and other markets.  

The [QuickFIX/J protocol](https://www.quickfixj.org/) is chosen to be used in the Mini Exchange project. QuickFIX/J is a full-featured messaging engine for the FIX protocol. It is a 100% Java open-source implementation of the popular C++ QuickFIX engine and supports Java NIO asynchronous network communications for scalability and processing threading strategies. 

[FIX 4.2](https://www.fixtrading.org/standards/fix-4-2/), one of two most popular FIX protocol versions is selected for the project due to the wide range of its usage. Other versions can also be configured by using a different data dictionary file at the gateway. 

### UDP Connection
The User Datagram Protocol (UDP) is one of the main protocols of the Internet protocol suite. The UDP can be unicast, multicast, and broadcast. UDP provides the lowest latency among the internet protocol suite since UDP uses a simple connectionless communication model with a minimum of protocol mechanisms. Since UDP has no handshaking dialogues, it is not as reliable as TCP connections. Since UDP exposes the user’s program to the underlying network, delivery, ordering, or duplication protection is not guaranteed. Moreover, UDP is often used in time-sensitive applications due to avoid waiting for packets that are delayed due to retransmission and dropping packets. Fortunately, [Aeron](https://github.com/real-logic/aeron) has already taken care of these and provides a reliable messaging interface over UDP.  

Aeron is chosen to be used in the Mini Exchange project. According to the creator of Aeron, it is
> a messaging solution for efficient reliable UDP unicast, UDP multicast, and IPC message transport. 

### Aeron
[Aeron](https://github.com/real-logic/aeron) is an [OSI Layer 4 message transport protocol](https://en.wikipedia.org/wiki/OSI_model#Layer_4:_Transport_Layer) maintained by Real Logic for efficient, reliable UDP unicast, UDP multicast, and IPC message transport.  Aeron has the advantage of high bandwidth with low latency, reliability, flow control, and congestion control. Sending requests and receiving responses using Aeron needs two connections since Aeron uses unidirectional connections.  

Media Driver in Aeron handles all media-specific logic in use for any active publications or subscriptions. It consists of the Driver Conductor, Receiver, Sender, and Client Conductor. Media Driver has four options of threading modes: Dedicated, Shared Network, Shared, and Invoker. We used dedicated mode for the most performance benefit.  

In this Mini Exchange project, the Aeron messaging system is implemented in Java. Since Aeron provides easy-to-use and efficient and reliable UDP multicast, it is selected as the library to use for communication between gateway, order matching engine, and tickerplant. We also explored alternative messaging libraries such as Kafka. However, it does not provide means of sending messages via multicast which is essential for ensuring fairness in financial systems.

### Agrona
[Agrona](https://github.com/real-logic/agrona) is a library also maintained by Real Logic that provides efficient data structures and utility methods commonly deployed in high performance Java programs because it uses off-heap memory that is not subjected to garbage collection by the JVM. Various Agrona data structures were implemented in Aeron application, such as Agrona buffers and hashmap.

The Agrona buffer is used in our system to store information sent through Aeron stream channels because Aeron is designed to be used in conjunction with Agrona data structure.  The Agrona hashmap collection is used in both tickerplant and matching engine to handle stock symbols-to-orderbooks mapping. Its [boxing prevention](https://aeroncookbook.com/agrona/data-structures/) (Preventing conversion from a Java primitive type to its wrapper classes) is the reason why we chose it over Java hashmap.

### IEX Market Data (DEEP)
[IEX DEEP](https://iextrading.com/docs/IEX%20DEEP%20Specification.pdf) is the official format by the IEX to display changes in each price level, such as share quantity at price and update process status, in a binary format. The binary format facilitates messaging efficiency and is easy to parse compared to other market data formats such as FIX which is ASCII encoded. For those reasons, the team planned to have tickerplant multicasting price level update messages in IEX format to assigned subscribers, which is not included but can be easily added by incorporating another Aeron multicast publisher.  

### Vagrant
[Vagrant](https://www.vagrantup.com/) is software that is aimed at building and managing multiple virtual machine environments in a single system. Examples of the virtual machine environment that Vagrant could manage are VirtualBox, KVM, Hyper-V, Docker containers, VMware, and AWS. Vagrant has a benefit that increases development productivity by simplifying configuration management. Virtualization allows easy dependency management across different developers and easy scripting for DevOps.  

Due to those advantages, Vagrant is used for Mini Exchange to run the multiple clients, multiple order matching engines, gateway, and tickerplant each in a separate virtual machine environment.  

[Docker](https://www.docker.com/?utm_source=google&utm_medium=cpc&utm_campaign=search_na_brand&utm_term=docker_exact&utm_content=modern&gclid=CjwKCAjw7IeUBhBbEiwADhiEMYAjAqEfem8-p7_q9hNu6R-iUufXZjm6DFVFGQqBFtxgcZ-P2_L2bxoC878QAvD_BwE), the OS-level virtualization software to deliver software in containers, was also considered to use instead of VirtualBox.  Docker uses containers to create user defined environment to build a specific application, which is more light-weighted compared to the virtual machines. However, the team intends to simulate real world scenario where different components of the exchange is hosted in different machines. Therefore, virtual machines are used to run the entire application. In addition, Docker does not support multicast traffic over its bridged network which is the default networking mode among all containers. However, we can easily configure multicast in VMs by adding multicast addresses (typically in the form of 224.x.x.x) to the routing table.

### JUnit Testing
[JUnit](https://junit.org/junit4/javadoc/4.12/org/junit/Test.html) is a testing utility that facilitates writing methods specifically for testing the correctness of the behavior of java code. The team splits exchange components into multiple parts where each team member develops these parts separately until final integration. JUnit testing enables team members to write test cases specifically for their classes to ensure expected code behaviors before the integration.

### Websocket
In order for the visualization dashboard to receive streaming data, the tickerplant also keeps a websocket server that publishes the same information regarding price level updates and execution details by using [Java-Websocket](https://github.com/TooTallNate/Java-WebSocket).



### Perspective
[Perspective](https://github.com/finos/perspective) is an interactive visualization library and it is designed to handle visulizing big data set. Originally developed at J.P. Morgan and open-sourced through the [Fintech Open Source Foundation (FINOS)](https://www.finos.org), Perspective allows users to configure data visualization and analysis entirely on browser, or in Python and [Jupyterlab](https://jupyterlab.readthedocs.io/en/stable/). 

Perspective enables real-time visualization of streaming data by using Apache Arrow for fast data streaming. It maintains SQL-like tables that will be constantly updated and can be used at either the server-side or the client-side or both. We have used the client-server replicated setup because it allows better performance when multiple clients are connected to the same dashboard. 

In order for the market data published from the tickerplant to be properly handled by Perspective, we need to stream it using Websocket and therefore we also include a Websocket server in the tickerplant that stream orderbook update information in JSON for Perspective to process and visualize.

The primary motivation for choosing perspective as the orderbook visualization tool is that it is supported by [FINOS(Fintech Open Source Foundation)](https://www.finos.org) as a financial data visualization tool and is well-suited for our purpose. Its open-source nature makes our visualization interface code accessible by people in the financial field who are not professional programmers. Also, there are existing video tutorials and examples in [perspective github repo](https://github.com/finos/perspective) that facilitate users to learn how to use Perspective API quickly.

## Component

![architecture diagram](exchange.png)

### Client

#### Description
The Client application (aka Trader) is used to send order to gateway via TCP in proper FIX format. They can send new order, market order and limit order, and order cancel request. Furthermore, multiple clients can connect to the same gateway via TCP connection in different FIX sessions. After submitting orders, clients will receive execution report from the gateway where they are informed about whether their trade is accepted, executed, canceled, or rejected.
#### Implementation
The session of the Client application is set up by configuration in the SessionSettings class and setting a Client class as the application. A socket initiator for the TCP connection is created based on the session settings. A client first logs on, establish a FIX session with the gateway socker acceptors, and submit orders. Then, the client will specify clOrdId for new order or cancel request for its own tracking purpose (e.g. client will submit cancel request by referring to the id of the order to be cancelled). Client application could also send the multiple orders by reading from a script containing order instructions on each line by using the ClientMessageParser class, which stores each single order in an ArrayList. ClientMessageParser class also classifies the String in to certain tag format of FIX.  
#### What could be improved
One of the aspects that needs to be improved for client was enabling the feature of users able to manual entering the trading order in the User Interface. Currently, users can change the textfile to enter the orders. However, this is not able to be done in the frontend user interface. 
Currently, the dashboard is connected with python-based AI client that sends FIX message to the gateway. For future expansion, the Client Application would be connected with the dashboard to be able to manually enter the trading order in the dashboard. This could be modifying server.py and index.js while connecting to client application.


### Gateway
#### Description
The Gateway application communicates with Client application and order matching engines. A gateway allows clients to submit orders to the exchange and receive updates regarding their orders and route the trade requests to the order matching engine that handles the specified instrument.
#### Implementation
The gateway is configured as a socket acceptor that establishes a new session for each connection from clients. Once receiving FIX requests from clients, the gateway then publishes the trade request message to the matching engine that is responsible for that specific instrument via an Aeron publication channel. Gateway will also perform fundamental order validation (e.g. whether the instrument is currently traded at the exchange, whether order type is valid, whether the cancel request is referring to an order that is never submitted for that client) and reject the order if invalid, and send the execution report back to the client containing order status and execution quantity and price if any. 

#### What could be improved
Currently, the gateway only allows connections from known client entities, i.e. those senderCompId that are included as a part of the configuration file. However, we could allow dynamic session that can allow the gateway to accept client IP addresses according to a specified pattern or users that exists in the User Table of the trading application.

### Order Matching Engine
#### Description
The order matching engine is a primary component of an exchange. It maintains an orderbook for every traded instrument at the exchange and try to match orders that are submitted by traders. An orderbook consists of all LIMIT orders that are not matched sorted in price and time priority, i.e. all bid orders are sorted ascendingly and all asks are sorted descendingly and if there are multiple orders at the same limit price, those submitted first will be matched first. There is a match if the bid price is greater than or equal to the ask price. If there are matches for an incoming order, then the orders that are matched against are removed from the orderbook if they are fully filled. If an order is partially filled, then that order's remaining quantity that is still open for matching is reduced by the filled quantity. Different order types are processed in different priorities. CANCEL orders are processed first followed by MARKET orders and finally LIMIT orders. MARKET orders don't have a specified price and will typically be matched at multiple price levels until their specified quantity is entirely filled. They can also be thought of a special case of LIMIT orders where the limit price for bid and ask are infinity and zero respectively.
#### Implementation
For fast matching, the orderbook must support constant time deletion since only order id will be specified in CANCEL request. In addition, a pointer to each price level must be maintained since we need to frequently add to the back of a queue of all existing orders for that price level when a new LIMIT order is added. We used LinkedHashmap as the queue for each price level since it maintains insertion orders and supports constant time deletion given key. Therefore, we use a Java TreeMap to maintain orders sorted by their limit prices and two additional hashmaps (orderId -> order and price -> pointer to the queue for that price level) for fast operations. Finally, after matching, execution reports containing the order status and price level changes will be sent by a UDP multicast message via Aeron to all gateways and tickerplant.
#### What could be improved
Unfortunately, Java Treemap is a Red-black tree implementation and does not expose TreeNode API publicly. Therefore, even though we provide the pointer to the tree node that contains the order, the remove operation will still take O(log n) where n is the number of price levels. For more efficient orderbook, we must implement a custom tree-based data structure that maintains pointers to ancestor nodes so that we could remove a node in constant time given a pointer. In addition, all matching for all symbols is processed in a single thread. Since orderbooks are independent of each other, the matching algorithm can be easily parallelized by using one thread per orderbook.


### Tickerplant
#### Description
The tickerplant is responsible for tracking the global state of each price level in the market upon receiving information from the order matching engine. The state information includes price, number of shares at that price level, timestamp when this price level is last updated, and the symbol of the stock. The full snapshot of the tickerplant is an level 2 orderbook where depth of the orderbook is shown, but information regarding specific order will not be displayed. Upon receiving messages from the matching engines, the tickerplant will update price levels on its own orderbooks and send price level update messages to all subscribers via multicast.
#### Implementation
The tickerplant is implemented as a String to Object HashMaps where each symbol is mapped into its corresponding tickerplant orderbook

The orderbooks at the tickerplant are implemented as a Java TreeMap where the key is price and the value is the size of all orders at that price level. The bid side treemap is sorted in descending order so the highest bid (best bid) is output as the first entry. The ask treemap is sorted in ascending order so the lowest ask (best ask) is output as the first entry.

The TreeMap update method is called when the tickerplant receives multicast messages from matching engines. After the update is completed, the tickerplant will send price level update messages as JSON to the tornado server that is responsible for updating tables used by Perspective through Java websocket which will then be used by our dashboard to update visualization plots.

#### What could be improved
The tickerplant maintains orderbooks for different stock symbols, and each orderbook contains two Java [TreeMaps](https://docs.oracle.com/javase/8/docs/api/java/util/TreeMap.html), one for the ask and one for the bid sides. The TreeMaps are maintained in a seperate class from the main orderbook class. The fact that individual side books are maintain in a separate class means more interclass interactions, which increases load during program execution. The orderbook class and side book class can potentially be integrated into one class to reduce call stacks during updates, enhancing the efficiency of per message price level update and information retrieval. 


### Dashboard

Dashboard provides an interactive visualization and analytics user interface using Perspective library where users can create different views (line plots, heatmaps, data grids, or custom elements) for the same stream of data and query them using SQL like manners. All data received from the dashboard is streamed through websocket. Our dashboad contains a websocket server and a javascript web client.

The server updates the orderbook table upon receiving JSON messages from the tickerplant and directly publishes them using its own tornado websocket server for Perspective to visualize. Note that currently the server is only acted as a relay between the tickerplant and the frontend but can be used to handle requests for additional websocket endpoints for order submission directly from the dashboard. 

The client mirrors the table from the server, format information, and display visualization in the frontend. The front end supports data filtering through user interaction with the interface. 



## Git Repository Layout



    group_03_project
    ├── Client                    
    │   ├── src/main/java        
    │   │     ├── Client.java                      # FIX Client Socket Initiator
    │   │     ├── ClientApp.java                   # FIX Client Main Entry
    │   │     └── ClientMessageParser.java         # Trade Message Parser
    │   │    
    │   └── pom.xml              
    │   
    ├── Gateway                    
    │   ├── src/main/java        
    │   │     ├── Gateway.java                     # FIX Gateway Socket Acceptor
    │   │     └── GatewayApp.java                  # FIX Gateway Main Entry
    │   │
    │   └── pom.xml 
    │
    ├── MatchingEngine                    
    │   ├── src/main/java        
    │   │     └──MatchingEngine.java               # Order Matching Engine
    │   │    
    │   └── pom.xml 
    │
    ├── MediaDriver                    
    │   ├── src/main/java        
    │   │     ├── AeronUtil.java                   
    │   │     ├── BasicMediaDriver.java            # Basic MD configuration
    │   │     ├── LowLatencyMediaDriver.java       # Low Latency MD configuration
    │   │     ├── Publisher.java                   # Wrapper Class for Publication
    │   │     └── Subscriber.java                  # Wrapper Class for Subscription
    │   │    
    │   └── pom.xml 
    │
    ├── OrderBook                    
    │   ├── src/main/java        
    │   │     ├── BookLevel.java
    │   │     ├── Order.java
    │   │     ├── OrderBook.java                   # Orderbook Data Structure
    │   │     ├── Report.java                      # Execution Report Published by ME
    │   │     └── TradeRequest.java                # Trade Request Message Sent by GW
    │   │
    │   └── src/test/java
    │         └── OrderBookTest.java    
    │
    ├── TickerPlant                    
    │   ├── src/main/java        
    │   │     ├── BookSide.java                    # Containers for BID & ASK OrderBooks
    │   │     ├── ByteDecoder.java                 # Byte -> Primitive Types
    │   │     ├── ByteEncoder.java                 # Primitive Types -> Byte
    │   │     ├── MessageFromME.java               # ME Message for Unit Testing
    │   │     ├── OrderBookTp.java                 # OrderBook at TP
    │   │     ├── PriceLevel.java                  
    │   │     ├── StockPrice.java                 
    │   │     ├── TPServer.java                    # WebSocket connecting TP and FrontEnd
    │   │     ├── TickerPlant.java                 # Update TickerPlant Orderbook
    │   │     └── toPriceLevelUpdateMessage.java   # IEX Byte -> String
    │   │    
    │   └── src/test/java
    │         ├── OrderBookTPTest.java             # Test Orderbook Update Logic 
    │         ├── PriceLevelTest.java              # Test Price Level Instantiation
    │         ├── PriceLevelUpdateMessageTest.java # Test IEX Decode/Encode
    │         └── StockPriceTest.java              # Test Stock Price Comparator
    │
    ├── dashboard  
    │   ├── initiator
    │   │     ├──order_scripts
    │   │     │          ├──test_scripts1.txt      # Order Submission Scripts
    │   │     │          └──test_scripts2.txt
    │   │     ├──spec
    │   │     │    ├──FIX42.xml                     # Different FIX Dictionaries
    │   │     │    └──...
    │   │     │     
    │   │     ├── application.py                   
    │   │     ├── client.py                        # FIX Dumb AI Trader in Python
    │   │     ├── client1.cfg
    │   │     ├── client2.cfg
    │   │     └── start.sh                         # Script to Launch Multiple Clients
    │   │
    │   ├── src 
    │   │     ├── client
    │   │     │     ├── index.html
    │   │     │     ├── index.js                   
    │   │     │     └──  index.less
    │   │     │      
    │   │     └── server                           
    │   │          ├──_init_.py
    │   │          └── server.py                   # Tornado Websocket Server
    │   │    
    │   ├── package.json            
    │   │                              
    │   ├── webpack.config.js
    │   │
    │   └── yarn.lock 
    │
    ├── scripts                    
    │   ├── client_install.sh            
    │   ├── install.sh
    │   ├── set_vagrant_env.sh
    │   └── start_all.sh
    │                   
    ├── Vagrantfile                    
    │
    └── pom.xml


The Exchange components are Client, Gateway, MatchingEngine, MediaDriver, OrderBook, TickerPlant, and dashboard. They are seperate directories at the root level of the repository. 

The scripts directory contains shell command to setup running applications and install all dependencies. 

The miscellaneous documents demonstrating high-level design concepts are also at the root of the repository.

The entire application is run on multiple separate virtual machines set up by the Vagrantfile on [VirtualBox](https://www.virtualbox.org/). Sufficient amount of RAM is required for running multiple VMs smoothing (at least 16G and 32G is recommended).


## Project Instruction
```bash=
# after git cloning
vagrant up # this might take up to 30 min since some dependencies have long build time such as quickfix-python
```
```bash=
# In another terminal
vagrant ssh matching-engine1
cd /vagrant && mvn -pl MatchingEngine -am clean package && java --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens jdk.unsupported/sun.misc=ALL-UNNAMED -jar MatchingEngine/target/MatchingEngine-1.0-SNAPSHOT-jar-with-dependencies.jar 192.168.0.51 1
```

```bash=
# In another terminal
vagrant ssh gateway
cd /vagrant && mvn -pl Gateway -am clean package && java --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens jdk.unsupported/sun.misc=ALL-UNNAMED -jar Gateway/target/Gateway-1.0-SNAPSHOT-jar-with-dependencies.jar 192.168.0.101 1
```

```bash=
# In another terminal
vagrant ssh tickerplant
cd /vagrant && mvn -pl TickerPlant -am clean package && java --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens jdk.unsupported/sun.misc=ALL-UNNAMED -jar TickerPlant/target/TickerPlant-1.0-SNAPSHOT-jar-with-dependencies.jar 192.168.0.201 1
```

```bash=
# In another terminal
vagrant ssh client
cd /vagrant && (sh initiator/start.sh 1 > /dev/null 2>&1 & yarn start)
```

Go to https://localhost:8080 to see the dashboard.

If using multiple matching engine, then after vagrant up
```bash=
# In another terminal
vagrant ssh matching-engine1
cd /vagrant && mvn -pl MatchingEngine -am clean package && java --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens jdk.unsupported/sun.misc=ALL-UNNAMED -jar MatchingEngine/target/MatchingEngine-1.0-SNAPSHOT-jar-with-dependencies.jar 192.168.0.51 1
```

```bash=
# In another terminal
vagrant ssh matching-engine2
cd /vagrant && mvn -pl MatchingEngine -am clean package && java --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens jdk.unsupported/sun.misc=ALL-UNNAMED -jar MatchingEngine/target/MatchingEngine-1.0-SNAPSHOT-jar-with-dependencies.jar 192.168.0.52 2
```

```bash=
# In another terminal
vagrant ssh gateway
cd /vagrant && mvn -pl Gateway -am clean package && java --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens jdk.unsupported/sun.misc=ALL-UNNAMED -jar Gateway/target/Gateway-1.0-SNAPSHOT-jar-with-dependencies.jar 192.168.0.101 1 2
```

```bash=
# In another terminal
vagrant ssh tickerplant
cd /vagrant && mvn -pl TickerPlant -am clean package && java --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens jdk.unsupported/sun.misc=ALL-UNNAMED -jar TickerPlant/target/TickerPlant-1.0-SNAPSHOT-jar-with-dependencies.jar 192.168.0.201 1 2
```

```bash=
# In another terminal
# If using order submission script and not using the dashboard
vagrant ssh client
cd /vagrant && python3.9 client.py client1.cfg -s order_scripts/test_script2.txt > /dev/null 2>&1 
```



## Testing

Integration test
- Due to time limits, integration test is currently only validated by manual inspection by sending orders in scripts and validate system states. Some helper methods are provided such as orderbook pretty formatter that can allow more human readable format.

Unit test
- Unit testing is carried out by using JUnit. Standard IDE such as Eclipse and Intellij IDEA have good support for this.
- For core components e.g. orderbooks, extensive unit tests cases are written involving checking the consistent state of orderbooks, such as checking if the total volume at a price level is equal to the sum of the volume of each individual orders on that level. Edge cases, such as empty orderbook, are also tested. 

## Result
The project is capable of establishing multiple client connections at gateway, submit orders either programmatically or in a order script, processing LIMIT, MARKET, and CANCEL orders at multiple matching engines, and publishing market data from tickerplant and finally using a web dashboard to visualize the state of the market.

## Example Output

![demo](orderbook-realtime.gif)

In addition to the market data dashboard, we can also use wscat to inspect the market data that is streamed from the tickerplant by running the following command on the host machine.

```bash=
# install wscat if not already by
# npm install -g wscat
wscat --connect ws://localhost:8081
```
![websocket](websocket.png)