# Package Operator Enhancements

Enhancement tracking repository for the Package Operator project.

Inspired by [Kubernetes enhancements](https://github.com/kubernetes/enhancements) process.

This repo contains enhancement issues. These issues are umbrellas for new enhancements to be added to Package Operator. An enhancement usually takes multiple releases to complete. And an enhancement can be tracked as backlog items before work begins. An enhancement may be filed once there is consensus in at least one Istio working group.

## Is My Thing an Enhancement?

An enhancement is anything that:
- a blog post would be written about after its release (ex. minikube, StatefulSets, rkt container runtime)
- will be graduating from one stage to another (ex. alpha to beta, beta to GA)
- needs significant effort or changes Package Operator in a significant way (ex. something that would take 10 person-weeks to implement, introduce or redesign a system component, or introduces API changes)
- impacts the UX or operation of Package Operator substantially such that engineers using Package Operator will need retraining
- users will notice and come to rely on

It is unlikely an enhancement if it is:
- fixes a bug
- fixing a flaky test
- refactoring code
- performance improvements, which are only visible to users as faster API operations, or faster control loops
- adding error messages or events

## When to Create a New Enhancement

Create an enhancement here once you:

- have circulated your idea to see if there is interest
- (optionally) have done a prototype in your own fork
- have identified people who agree to work on and maintain the enhancement
