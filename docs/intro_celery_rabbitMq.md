
# Introduction to Celery

- Celery is an open-source distributed task queue system that is widely used for building asynchronous and distributed applications.
- It allows you to execute tasks (functions or methods) asynchronously, making it particularly useful for tasks that are time-consuming, computationally intensive, or need to be scheduled for future execution.

### Key features of Celery
- Task scheduling and execution.
- Distributed task processing with multiple worker nodes.
- Support for task prioritization and retries.
- Scalability for handling large workloads.
- Monitoring and management tools.

### Example
- Consider a Django package, it has many tasks running simultaneously.
- If we wait for one task to complete before picking the next one, it will significantly increase the page loading time.
- To avoid this, we can run the tasks in parallel. This can be achieved by using Celery.

![Celery Example](./celery_png.png)

### What we need to use Celery
- To use Celery we need a message transport, as Celery can only create messages and keep them ready to be transported.
- These transporters are called as Brokers, The RabbitMQ is the well known broker transports.


# Introduction to RabbitMQ
- RabbitMQ is an open-source message broker software that facilitates communication between different parts of a distributed system.
- It is a key component of many modern software architectures, particularly in systems where different parts need to exchange data and messages in a scalable and asynchronous manner.
- RabbitMQ is feature-complete, stable, durable, and easy to install. It’s an excellent choice for a production environment.

### Key features of RabbitMQ
- Reliability
- Flexible Routing
- Clustering
- Highly Available Queues
- Management UI

### How RabbitMQ and Celery works
![RabbitMQ Example](./rabbitMqCeleryExample.png)

1. Install RabbitMQ by using the following commands:
```sudo apt-get install rabbitmq-server```
2. Celery is on the Python Package, so we can install it using pip
3. Create a python script for celery to perform a simple task
4. Now we can use the below command to executing our program with the worker argument
   






