# Notes for Microservices Talk


## Pro

Independent development.

Asynchronous lifecycles.

Can rewrite a microservice.

Can repurpose a microservice.

Can use a different technology stack in a microservice.

Horizontal scaling (ms necessary but not sufficient)

Feature scaling.

UI/UX can span services and domains - task oriented.

Persistance and data models can be split among domains.

## Con

More processes, containers, hosts to deploy, configure, monitor and manage. 

More interfaces to design implement and maintain.

Distributed systems are hard. Partial failures, single point of failure.

Must identify service scope and boundaries early.

## Arguments

Use the tools of devops.  Getting better all the time.

Exactly how would one modularise a monolith?  OSGI, really?

Even the simplest 3 tier system is distributed already.  (Single user desktop apps not an important category) A modern application can be viewed as a specialised communication medium for its users. 

Identify services by reducing size to surface/volume limit.  Start small and amalgamate.

## Distributed Systems

Don't pretend its not distributed.

Don't expect to replace PCs with RPCs

The nature of time.

Partial failure.


## Topologies

destop application

3 tier monolithic

3 tier siloed

3 tier integrated UI 

3 tier shared db(s)

task vs. domain services

read vs. write paths

network of services/actors

technical layers - web proxy, orchestration, data cache, service bus

loci of serialisation, sharding vs vector clocks

clustering

the dream infrastructure

##

Highly available systems were exotic beasts, now expected.

## Refs

Fowler

Tilkov



http://blog.circleci.com/its-the-future/

So I just need to split my simple CRUD app into 12 microservices, each with their own APIs which call each others’ APIs but handle failure resiliently, put them into Docker containers, launch a fleet of 8 machines which are Docker hosts running CoreOS, “orchestrate” them using a small Kubernetes cluster running etcd, figure out the “open questions” of networking and storage, and then I continuously deliver multiple redundant copies of each microservice to my fleet. Is that it?

-Yes! Isn’t it glorious?

I’m going back to Heroku