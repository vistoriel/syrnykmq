# SyrnykMQ

SyrnykMQ is a NestJS addon that makes integrating with RabbitMQ (AMQP) idiomatic and simple. It provides decorators for declaring message handlers, a configurable module for connection setup, and pluggable serializer/deserializer logic.

SyrnykMQ wires RabbitMQ into NestJS apps using a familiar pattern: register the module with connection options, declare handler classes using decorators and let the library handle lifecycle, topology setup and message dispatch. It supports topic-style routing (with wildcard resolution), request/reply semantics and automatic discovery of handler classes.

## Highlights
- **NestJS-native**: uses Nest's discovery and metadata APIs to wire handlers automatically
- **Decorators**: `EventHandler`, `CommandHandler`, `QueryHandler`, and `HandlersGroup` to structure consumers
- **Param decorators**: `Content`, `Message`, `Controls` for handler arguments
- **Producer + Request/Reply**: `SyrnykmqProducerService.publish` and `request` for fire-and-forget and RPC-style calls
- **Topology management**: declare exchanges, queues and bindings via module options
- **Pluggable serialization**: default JSON serializer/deserializer with option to override

## Installation

Install the package and peer dependencies (Nest and AMQP clients are required):

```sh
npm install syrnykmq @nestjs/common @nestjs/core amqp-connection-manager amqplib
```

## Quick start

Register the module in your Nest application and provide connection/topology options.

```ts
import { Module } from '@nestjs/common';
import { SyrnykmqModule } from 'syrnykmq';

@Module({
  imports: [
    SyrnykmqModule.register({
      urls: ['amqp://localhost:5672'],
      exchanges: [{
        name: 'app',
        type: 'topic',
        default: true
      }],
      queues: ['my-queue'],
    }),
  ],
})
export class AppModule {}
```

*The module is created using Nest's `ConfigurableModuleBuilder` so it supports the usual `register`/`registerAsync`
patterns for configuration.*

### Define Handlers Group

```ts
import { HandlersGroup, EventHandler, QueryHandler, Content, Controls } from 'syrnykmq';

@HandlersGroup({ queue: 'my-queue', exchange: 'app' })
export class MyHandlers {
  @EventHandler('user.created')
  async onUserCreated(@Content() payload: Record<string, unknown>) {
    // fire-and-forget
    console.log('User created', payload);
  }

  @QueryHandler('user.get')
  async onUserGet(@Content('id') id: string, @Controls() controls) {
    // return a response for request/reply
    controls.setHeaders({ 'x-meta': 'example' });
    return { id, name: 'Alice' };
  }
}
```

### Fire events or requests

```ts
import { HandlersGroup, EventHandler, QueryHandler, Content, Controls } from 'syrnykmq';

@Injectable()
export class UserService {
  constructor(private producer: SyrnykmqProducerService) {}

  async getUser(id: number) {
    // request user and wait for response
    const user = await this.producer.request('app', 'user.get', { id });
    return user;
  }

  async emitUserRegistered(user: User) {
    // fire-and-forget event
    this.producer.publish('app', 'user.registered', user);
  }
}
```

- `@HandlersGroup({ queue?, exchange? })` — marks a class as a message handlers group and can set defaults for the group.
- `@EventHandler(patterns, options?)` — register an event handler (no reply expected).
- `@CommandHandler(patterns, options?)` — register a command handler (reply expected).
- `@QueryHandler(patterns, options?)` — register a query handler (reply expected).

Parameter decorators (for handler method arguments):

- `@Content()` — extracts and optionally selects a property from the deserialized message content.
- `@Message()` — injects the raw AMQP `ConsumeMessage` object.
- `@Controls()` — injects controls you can call to `ack`, `nack` or `setHeaders` on the response.

## Module options

The `SyrnykmqModuleOptions` object supported when registering the module contains the following fields:

- `urls: string[]` — required AMQP connection URLs
- `reconnectTimeInSeconds?: number` — reconnect interval (default 5s)
- `heartbeatIntervalInSeconds?: number` — AMQP heartbeat (default 5s)
- `connectionOptions?: AmqpConnectionOptions` — passed to the underlying connection manager
- `logger?: LoggerService` — provide a Nest `LoggerService` implementation
- `exchanges?: Exchange[]` — exchanges to assert Each exchange: `{ name, type, default?, bindings? }`.
- `queues?: (Queue | string)[]` — queues to assert. `Queue` may have `{ name, default?, bindings?, useDefaultDLX? }`
- `resolveTopics?: boolean` — when `true` (default), patterns are matched using topic-style wildcards (`*`, `#`); when `false` exact matches are used
- `autoAck?: boolean` — when `true` messages are automatically acknowledged after handler success (default: `true`)
- `serializer?: SyrnykmqSerializer` — function `(object) => Buffer` used when sending messages
- `deserializer?: SyrnykmqDeserializer` — function `(buffer) => object` used when receiving messages

Defaults used by the implementation:

- JSON serialize/deserialize are used by default: `defaultSerializer` and `defaultDeserializer`.
- Default reconnect and heartbeat times are 5 seconds.

## Serialization

By default SyrnykMQ serializes with `JSON.stringify` and parses messages with `JSON.parse`. You can override both
`serializer` and `deserializer` in module options to support binary formats or custom framing.

## Exceptions

The library exposes a few custom exceptions used at runtime:

- `FailedConnectToBrokerException` — thrown when initial connection fails.
- `FailedReceiveResponseException` — thrown when a request times out waiting for a reply.
- `FailedResponseException` — thrown when a response contains an error header.
- `NotSetDefaultExchangeException` — thrown when code expects a default exchange but none was set.
- `NotSetDefaultQueueException` — thrown when code expects a default queue but none was set.

## Request / Reply details

The `request` flow uses `amq.rabbitmq.reply-to` and correlation IDs. The producer waits for the first reply message
with the matching correlation ID and will throw `FailedReceiveResponseException` if the reply doesn't arrive in time.

## Contributing & development

- Build: `npm run build`
- Format: `npm run format`
- Lint: `npm run lint`

If you find issues or want to propose features, please open an issue or PR.
