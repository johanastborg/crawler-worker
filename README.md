# crawler-worker

A Clojure-based web crawler worker that consumes URLs from a RabbitMQ queue, crawls them, extracts new links, and publishes the discovered links back to RabbitMQ.

## Description

`crawler-worker` is designed to be part of a distributed crawling system. It utilizes:
- **RabbitMQ** for task distribution and result collection.
- **Java NIO** (`AsynchronousSocketChannel`) for high-performance, non-blocking HTTP requests.
- **Clojure `core.async`** for managing concurrency and data flow.

The worker performs the following steps:
1. Listens for messages (URLs) on a RabbitMQ queue.
2. Connects to the target URL asynchronously.
3. Downloads the content.
4. Parses the content to find other links (`<a href="...">`).
5. Batches the discovered links and publishes them back to a RabbitMQ exchange.

## Prerequisites

- **Java JDK** (version 7 or higher recommended, given the code age)
- **Leiningen** (for dependency management and running the project)
- **RabbitMQ Server**

## Configuration

**Important:** The current version of the code has a hardcoded RabbitMQ host address.

Before running, you must edit `src/crawler_worker/core.clj` to point to your RabbitMQ instance.

Open `src/crawler_worker/core.clj` and find the following line:

```clojure
(def rabbit-conn (rmq/connect {:host "ec2-54-213-238-4.us-west-2.compute.amazonaws.com"}))
```

Change `"ec2-54-213-238-4.us-west-2.compute.amazonaws.com"` to your RabbitMQ host (e.g., `"localhost"` or your server IP).

## Usage

1. **Start RabbitMQ**: Ensure your RabbitMQ server is running.

2. **Run the Worker**:
   From the project root directory, run:

   ```bash
   lein run
   ```

   The worker will connect to the RabbitMQ exchange `url-crawler`, bind to the queue, and start listening for messages with the routing key `to-crawl-urls`.

## Architecture

- **Exchange**: `url-crawler` (Topic exchange)
- **Input Routing Key**: `to-crawl-urls`
- **Output Routing Key**: `discovered-urls` (Published with type `quote.update`)

### Workflow

1.  **Consumer**: `create-topic-channel-reader` creates a consumer that listens on the `url-crawler` exchange.
2.  **Processing**: When a URL is received, `crawl-url` is triggered.
3.  **Crawling**: `crawl-url` opens an asynchronous socket connection to port 80 of the host.
4.  **Extraction**: `ach-reader` reads the response and applies a regex to find links.
5.  **Buffering & Publishing**: `read-and-send` buffers discovered URLs until the buffer size exceeds ~512KB, then publishes the batch to RabbitMQ.

## License

Copyright © 2014

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
