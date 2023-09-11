
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
- Consider a Django package; it has many tasks running simultaneously.
- If we wait for one task to complete before picking the next one, it will significantly increase the page loading time.
- To avoid this, we can run the tasks in parallel. This can be achieved by using Celery.

![alt text](./celery_png.png)


# Introduction to RabbitMQ
- RabbitMQ is an open-source message broker software that facilitates communication between different parts of a distributed system.
- It is a key component of many modern software architectures, particularly in systems where different parts need to exchange data and messages in a scalable and asynchronous manner. Here's a brief overview of RabbitMQ

### Why we need RabbitMQ
- Celery requires a service to send and receive messages; usually, this comes in the form of a separate service called a message broker.
- 
