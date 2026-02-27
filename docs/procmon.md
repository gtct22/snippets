# Process Monitor

## Overview
**Process Monitor** is an Azure Function App that forms part of the monitoring and alerting layer for the Client Onboarding process.

It provides:

* Automated health monitoring of application services
* Shutdown of message processing when dependencies are unhealthy
* Event-driven alerting via PagerDuty for errors

## Purpose

The Process Monitor ensures:

1. The onboarding pipeline does not continue processing when dependent services are in a degraded or unhealthy state.
2. Operational teams are alerted immediately when critical onboarding errors occur.



## Architecture

The Function App contains two functions:

### Timer Trigger – Health Check Monitor

**Responsibility:** Monitor service health and stop onboarding when dependencies are unhealthy.

#### Behaviour

* Executes on a schedule.
* Calls health check endpoints for all adapters/engines within the Client Onboarding stack.
* Evaluates each response.

If any service reports:

* `Degraded`
* `Unhealthy`

The function will:

* Disable RabbitMQ, preventing further onboarding message processing.

RabbitMQ is not automatically re-enabled if services later return to a healthy state.
Re-enablement is a manual activity.

#### Health Check Monitor Flow
```
Timer Trigger
    ↓
Call Adapter/Engine Health check Endpoints
    ↓
Evaluate Health Status
    ↓
If Degraded/Unhealthy?
    ↓
Disable RabbitMQ
```

---

### Service Bus Trigger – Integration Error Event Monitor

**Message Source:** COB Engine

#### Behaviour

* Listens for integration error event messages published by the COB Engine.
* Filters incoming messages for relevant error types.

**Currently supported error types:**

* Avaloq-related errors

When a relevant error is detected:

1. The message is transformed into a PagerDuty-compatible payload.
2. The error is assigned a specific PagerDuty error group based on the error message content.
3. The payload is sent to PagerDuty.

Non-relevant messages are ignored currently.

##### Integration Error Event Flow

```
COB Engine processing error
    ↓
Publish Integration Error Event to Service Bus
    ↓
Process Monitor Triggered
    ↓
Filter Relevant Errors
    ↓
Transform to PagerDuty Payload
    ↓
Send to PagerDuty
```

## Other Considerations

### Manual RabbitMQ Re-Enablement

When services return to a healthy state RabbitMQ must be manually re-enabled.

* TODO How to re-enable RabbitMQ?

## Future Enhancements

* Automatic RabbitMQ reactivation after healthy state sustained for all services.
* Error filtering currently limited to Avaloq-related errors. This can be expanded to handle other error types within Client Onboarding.
* PagerDuty integration is direct via APIM, future plans may include a new service to function as a wrapper around PagerDuty integrations.
* Dashboard reporting.

