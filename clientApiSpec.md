# Participant API

## Rationale

The KZG trusted setup will benefit from have a diversity of approaches at all points in the application stack. We wish to encourage a variety of UI modules, covering multiple modes, hardware platforms, OS varieties, etc. This document is intended for those who wish to implement a client to perform the computation.

This API aims to be simple to implement yet sufficient to cope with the real-world variations that will be encountered.

## Reference

The REST end points noted in this document are described in detail here: [API Trusted Setup MPC](https://hub.apitree.com/stonebell/KZG-MPC-participant/)

## Overview

The participant client will follow a session life-cycle with these stages:

- Landing
- Authentication
- Wait in the lobby
- Queue
- Compute
- Post-contribution

## Landing

The starting point for the client need only provide a mechanism to initiate login. Additional, optional data could be presented, including overall ceremony data, queue length, etc. Such data can be retrieved using the `ceremony/status` end point.

## Authentication

Participants are required to identify themselves and pass some anti-sybil qualifying tests. The process follows common OAuth2 usage. 

The client must login using the `login` end point. A successful request will return with a token (JWT), which must be passed in subsequent requests (except where noted).

The JWT will remain valid for the duration of the session, i.e. until the client logs out, or closes the web page. 

## Waiting in the Lobby

### Participant



#### Request

The participant asserts their liveness (`lobby/checkin`). This confirms the participant's readiness to perform the computation at short notice, should it be required.

#### Response

The response will include these items:

- the participant’s status (`waiting` or `queued`)
- the current average duration of the wait 
- a deadline by which time the client must check in again

The client must call the `lobby/checkin` end point by the deadline or risk being marked as absent and removed from the queue. The response will indicate a new check-in deadline. 

The gap between check-in times will decrease as the head of the queue approaches. The client will loop (wait for the deadline; call `queue/checkin`; receive a new deadline) until the status indicates that it is the participant’s turn to contribute. The client must then move on to the computation phase.

Should the queue be empty when `queue/join` is called, the status will indicate that the computation is ready. No waiting is required. The client must move straight to the computation phase. 

A participant may leave the queue voluntarily by calling the `queue/leave` end point. Clients should issue this call if the user is intentionally leaving. The participant will be removed in any case once they fail to check in.

### Coordinator

The coordinator will track individual participants, their position in the queue, and their check-in deadlines. The queue will advance every time a computation is completed and verified. Should a participant fail to check in by their deadline (plus an allowance for latency, clock variations, etc), the participant will be removed from the queue. 

The expected time to participate will be estimated based on the average round trip computation time (including verification) times the number of waiters ahead in the queue. This will be recalculated with the latest data each time the participant checks in.

The initial check-in deadline will be 2 hours prior to the estimated start time. They will continue with check-ins at an interval of 1 hour or half the expected wait time, whichever is smaller. While the wait time is less than 1 hour, the deadline will be halved at each check-in call, to a minimum of 5 seconds. 

For the participant currently computing their contribution, the coordinator must allow enough time for slow contributors to complete while enforcing on a deadline so as to abort failed or abandoned computations. The deadline will be 3 minutes. During this time, the participant at the head of the queue will poll at 5 second intervals until either the computation is completed or the deadline is reached. 

## Joining the Queue

### Participant

#### Request

The participant first requests to join the queue (`lobby/enter`). 

#### Response

The response will include these items:

- the participant’s status in the queue
- the expected time at which to start
- a deadline by which time the client must check in again

The client must call the `queue/checkin` end point by the deadline or risk being marked as absent and removed from the queue. The response will indicate a new check-in deadline. 

The gap between check-in times will decrease as the head of the queue approaches. The client will loop (wait for the deadline; call `queue/checkin`; receive a new deadline) until the status indicates that it is the participant’s turn to contribute. The client must then move on to the computation phase.

Should the queue be empty when `queue/join` is called, the status will indicate that the computation is ready. No waiting is required. The client must move straight to the computation phase. 

A participant may leave the queue voluntarily by calling the `queue/leave` end point. Clients should issue this call if the user is intentionally leaving. The participant will be removed in any case once they fail to check in.

### Coordinator

The coordinator will track individual participants, their position in the queue, and their check-in deadlines. The queue will advance every time a computation is completed and verified. Should a participant fail to check in by their deadline (plus an allowance for latency, clock variations, etc), the participant will be removed from the queue. 

The expected time to participate will be estimated based on the average round trip computation time (including verification) times the number of waiters ahead in the queue. This will be recalculated with the latest data each time the participant checks in.

The initial check-in deadline will be 2 hours prior to the estimated start time. They will continue with check-ins at an interval of 1 hour or half the expected wait time, whichever is smaller. While the wait time is less than 1 hour, the deadline will be halved at each check-in call, to a minimum of 5 seconds. 

For the participant currently computing their contribution, the coordinator must allow enough time for slow contributors to complete while enforcing on a deadline so as to abort failed or abandoned computations. The deadline will be 3 minutes. During this time, the participant at the head of the queue will poll at 5 second intervals until either the computation is completed or the deadline is reached. 

## The Computation Phase

### Request

Participants will issue a `contribution/start` call to start the computation phase. This request must be issued within 15 seconds of the `queue/checkin` response that notifies the contributor to begin.

Any time-consuming actions required to derive randomness should have been done prior to sending this request, so that the contribution time is reduced to the minimum.

The request may include an array of tags to identify:
- the client implementation 
- the computation implementation to be used

### Response

The response will include the transcript from the latest valid contribution. 


Once the computation is complete, the updated transcript must be returned using a `contribution/complete` call. 

In the event that an error is caught preventing completion, the client must call to `contribution/abort`, and advise that they are aborting. This will allow the next participant to proceed with minimal delay. 

## Post-Contribution

Once the contribution has been returned, the client may perform additional tasks such as:

- Verification
- Capture and display of the contribution signatures.
- Submitting an attestation

These actions are not mandatory, although strongly recommended.

## Rate Limiting
- `login` requests will be limited to 50 per hour from a single IP address.
- Other requests without valid authentication will be limited to 100 per hour from a single IP address
- Other requests with valid authentication will be limited to 50 per minute from a single IP address