# Tor (Guest lecture)

A few years Nick Mathewson thought it was important to be able to build a system like Tor early-on, because it's a lot easier for people to ban an idea like Tor which hasn't been implemented yet and make claims against it, whereas if the technique has been made out of a prototype phase the discussion would be more on how to improve it.

From the start it was clear that security and usability are two major principles. For an anonymity-providing service like Tor it depends on operating on a wide group of users using the service to provide privacy.

## Design

Simplistic design:

U wants to send to S. It uses two routers `S_1` and `S_2`, which has their own pub-private key pairs. U will proceed to send to `S_1` the ciphertext `E_1(E_2(m))` where `m` is the message, and `E_1` and `E_2` are the public key of `S_1` and `S_2` respectively.

Increasing number of nodes:

In this example, if S_1 and S_2 is controlled then the entire network privacy guarantee goes down. However, if we increase this to be on the order of tens of *thousands* of routers, then it becomes much more resilient and robust.

It should be noted that with more users the amount of different traffic that each onion router see will increase, but unless you are able to see the network at a global adversary level most of this information won't mean anything to you.

## Drawbacks

- Global adversary can monitor exit node traffic and use it to perform traffic correlation (be able to detect that A is most likely talking to B). 
  - Dealing with this issue is difficult. We need to have a way of making the traffic across the entire Tor network more uniform (same amount of traffic + traffic pattern). 
  - This can be done with like cover traffic, randomly increasing delay, increasing overhead, etc. 
  - Research is still ongoing on these areas + quantifying which will work best to evade detection, but this is an ongoing problem for now.

