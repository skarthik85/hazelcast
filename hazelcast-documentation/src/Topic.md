## Topic

Hazelcast provides distribution mechanism for publishing messages that are delivered to multiple subscribers which is
also known as publish/subscribe (pub/sub) messaging model. Publish and subscriptions are cluster wide. When a member
subscribes for a topic, it is actually registering for messages published by any member in the cluster, including the
new members joined after you added the listener.

### Statistics

Topic has two statistic variables that can be queried. These values are incremental and local to the member.

```java

    final HazelcastInstance instance = Hazelcast.newHazelcastInstance(config);
    final ITopic<Object> myTopic = instance.getTopic("myTopicName");

    myTopic.getLocalTopicStats().getPublishOperationCount();
    myTopic.getLocalTopicStats().getReceiveOperationCount();


```

getPublishOperationCount returns total number of publishes from start of this node.
getReceiveOperationCount returns total number of received messages from start of this node.
Note that this values are not backed up, if node does down, this values will be lost.

These values can be also viewed from Management Center. See [Management Center]#topics
You can also close this feature with topic configuration. See [Configuration]#topic-configuration

### Internals

Each node has list of all registrations in the cluster. When a new node registered for a topic,
it will send a registration message to all members in the cluster. And also when a new node joins the cluster, it will
receive all registrations made so far in the cluster.

The behaviour of topic is very different for different values of configuration parameter `globalOrderEnabled`.

- `globalOrderEnabled` disabled.

    Messages are ordered, i.e. listeners (subscribers) will process the messages in the order they are actually published.
    If cluster member M publishes messages *m1*, *m2*, *m3*,...,*mn* to a topic **T**, then Hazelcast makes sure that
    all of the subscribers of topic **T** will receive and process *m1*, *m2*, *m3*,...,*mn* in the given order.

    Here is how it works.
    Lets says that we have three nodes, node1 , node2 and node3. And lets says that node1 and node2 are registered to
    a topic named `news`. Notice that all three nodes know that node1 and node2 registered to `news`.

    - Diagram1 showing registrations and nodes.

    In our example, node1 publishes two messages `a1` and `a2`. And node3 publishes two messages `c1` and `c2`.
    When node1 and node3 publishes a message, they will check their local list for registered nodes. Discovers that
    node1 and node2 are in the list. Then fires messages to those nodes. One of the possible order of messages
    are received can be following.

     -Diagram2
            Node1           Node2    Node3
             c1               c1
             b1               c2
             a2               a1
             c2               a2


- `globalOrderEnabled` enabled.

    When enabled, it guarantees that all nodes listening the same topic will get messages in the same order.

    Here is how it works.
    Lets says that we have three nodes, node1 , node2 and node3. And lets says that node1 and node2 are registered to
    a topic named `news`. Notice that all three nodes know that node1 and node2 registered to `news`.

     - Diagram1 showing registrations and nodes.

    In our example, node1 publishes two messages `a1` and `a2`. And node3 publishes two messages `c1` and `c2`.
    When a node publishes messages over topic `news`, it first calculates which partition `news` id correspond to. Then
    send an operation to owner of the partition for that node to publish messages. Lets assume that `news` corresponds
    to a partition that node2 owns. node1 and node3 first sends all messages to node2. Not that following diagram
    shows only one of the possibilities.

    - Diagram2 showing registrations and nodes.
            Node1           Node2       Node3
                              a1
                              c1
                              a2
                              c2

    Then node2 publishes these messages by looking at registrations in its local list. It sends these messages to node1
    and node2(it will make a local dispatch for itself).

    - Diagram2 showing registrations and nodes.
            Node1           Node2       Node3
             a1               a1
             c1               c1
             a2               a2
             c2               c2

    This way we guaranteed that all nodes will see the events in same order.

In both cases, there is a striped executor in EventService responsible in for dispatching the received message. For all
events in Hazelcast, the order that events are generated and the order they are published the user are guaranteed to
be same via this StripedExecutor. There are 'hazelcast.event.thread.count'(default is 5) threads in StripedExecutor.
For a specific event source(for topic, for a particular topic name), `hash of that sources name % 5` gives the id of
responsible thread. Note that there can be another event source (entryListener of a map, item listener of a collection etc)
corresponding to same thread. In order not to make other messages to block, heavy process should not be done in this thread.
If there is a time consuming work needs to be done, the work should be handed over to another thread. see
[this example]#topic-examples

### Topic Configuration

- From xml

    ```xml
    <hazelcast>
         ...

        <topic name="yourTopicName">
            <global-ordering-enabled>true</global-ordering-enabled>
            <statistics-enabled>true</statistics-enabled>
            <message-listeners>
                <message-listener>MessageListenerImpl</message-listener>
            </message-listeners>
        </topic>

         ...
    </hazelcast>
    ```

- Programatically

    ```java

    final Config config = new Config();
    final TopicConfig topicConfig = new TopicConfig();
    topicConfig.setGlobalOrderingEnabled(true);
    topicConfig.setStatisticsEnabled(true);
    topicConfig.setName("yourTopicName");
    final MessageListener<String> implementation = new MessageListener<String>() {
        @Override
        public void onMessage(Message<String> message) {
            // process the message
        }
    };
    topicConfig.addMessageListenerConfig(new ListenerConfig(implementation));
    final HazelcastInstance instance = Hazelcast.newHazelcastInstance(config)


    ```

Default values are

- Global ordering is false, meaning there is no global order guarantee by default

- Statistics are true, meaning statistics are calculated by default.

Topic related but not topic specific configuration parameters

    - "hazelcast.event.queue.capacity" : default value is 1,000,000
    - "hazelcast.event.queue.timeout.millis" : default value is 250
    - "hazelcast.event.thread.count" : default value is 5

For these parameters see [Distributed Event Config]#not-availaible-yet


### Examples

```java
import com.hazelcast.core.Topic;
import com.hazelcast.core.Hazelcast;
import com.hazelcast.core.MessageListener;
import com.hazelcast.config.Config;

public class Sample implements MessageListener<MyEvent> {

    public static void main(String[] args) {
        Sample sample = new Sample();
        Config cfg = new Config();
        HazelcastInstance hz = Hazelcast.newHazelcastInstance(cfg);
        ITopic topic = hz.getTopic ("default");
        topic.addMessageListener(sample);
        topic.publish (new MyEvent());
    }

   public void onMessage(Message<MyEvent> message) {
        MyEvent myEvent = message.getMessageObject();
        System.out.println("Message received = " + myEvent.toString());
        if (myEvent.isHeavyweight()) {
            messageExecutor.execute(new Runnable() {
                public void run() {
                    doHeavyweightStuff(myEvent);
                }
            });
        }
    }

    // ...

    private static final Executor messageExecutor = Executors.newSingleThreadExecutor();
}
```
