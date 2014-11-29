# Erlang bindings for NaCl

This library provides bindings for the NaCl cryptographic library for Erlang. Several such libraries exist, but this one is a re-write with a number of different requirements, and foci:

* Erlang/OTP 17.3. This library *needs* the newest dirty scheduler implementation.
* Uses the libsodium sources. This is a deliberate choice, since it is easy to implement.
* Does not provide anything but the original NaCl code base. This is also a deliberate choice so we are not in a situation where we can't jump to a newer version of the library later at some point.
* Tests created by aggressive use of Erlang QuickCheck.

This package draws heavy inspiration from "erlang-nacl" by Tony Garnock-Jones.

In addition, I would like to thank Steve Vinoski and Sverker Eriksson for providing the Dirty Scheduler API in the first place.

# TODO

* Write Eunit/Common Test cases which verifies that the byte-output of the functions matches the expected output from the NaCl library.

# Overview

The NaCl cryptographic library provides a number of different cryptographic primitives. In the following, we split up the different generic primitives and explain them briefly.

*A note on Nonces:* The crypto API makes use of "cryptographic nonces", that is arbitrary numbers which are used only once. For these primitives to be secure it is important to consult the NaCl documentation on their choice. They are large values so generating them randomly ensures security, provided the random number generator uses a sufficiently large period. If you end up using, say, the nonce `7` every time in communication while using the same keys, then the security falls.

The reason you can pick the nonce values is because some uses are better off using a nonce-construction based on monotonically increasing numbers, while other uses do not. The advantage of a sequence is that it can be used to reject older messages in the stream and protect against replay attacks. So the correct use is up to the application in many cases.

## Public Key cryptography

This implements standard Public/Secret key cryptography. The implementation roughly consists of two major sections:

* *Authenticated encryption:* provides a `box` primitive which encrypts and then also authenticates a message. The reciever is only able to open the sealed box if they posses the secret key and the authentication from the sender is correct.
* *Signatures:* allows one party to sign a message (not encrypting it) so another party can verify the message has the right origin.

## Secret key cryptography

This implements cryptography where there is a shared secret key between parties.

* *Authenticated encryption:* provides a `secret box` primitive in which we can encrypt a message with a shared key `k`. The box also authenticates the message, so a message with an invalid key will be rejected as well. This protects against the application obtaining garbage data.
* *Enryption:* provides streams of bytes based on a Key and a Nonce. These streams can be used to `XOR` with a message to encrypt it. No authentication is provided. The API allows for the system to `XOR` the stream for you while producing the stream.
* *Authentication:* Provides an implementation of a Message Authentication Code (MAC).
* *One Time Authentication:* Authenticate a message, but do so one-time. That is, a sender may *never* authenticate several messages under the same key. Otherwise an attacker can forge authenticators with enough time. The primitive is simpler and faster than the MAC authenticator however, so it is useful in some situations.

## Low-level functions

* *Hashing:* Cryptographically secure hashing
* *String comparison:* Implements guaranteed constant-time string comparisons to protect against timing attacks.

# Rationale

Doing crypto right in Erlang is not that easy. For one, the crypto system has to be rather fast, which rules out Erlang as the main vehicle. Second, cryptographic systems must be void of timing attacks. This mandates we write the code in a language where we can avoid cache timing attacks. This leaves only C as a contender, more or less. The obvious way to handle this is by the use of NIF implementations, but most C code will run to its conclusion once set off for processing. This is a major problem for a system which needs to keep its latency in check. The solution taken by this library is to use the new Dirty Scheduler API of Erlang in order to provide a safe way to handle the long-running cryptographic processing. It keeps the cryptographic primitives on the dirty schedulers and thus it avoids the major problem.

Focus has first and foremost been on the correct use of dirty schedulers, without any regard for speed. The plan is to extend the underlying implementation, while keeping the API stable. In a future version, we might want to make simple short-lived crypto-calls directly on the Erlang scheduler rather than moving these to a separate scheduler and paying the price of scheduler invocation.

Also, while the standard `crypto` bindings in Erlang does a great job at providing cryptographic primitives, these are based on OpenSSL, which is known to be highly problematic in many ways. It is not as easy to use the OpenSSL library correctly as it is with these bindings. Rather than providing a low-level cipher suite, NaCl provides intermediate level primitives constructed as to protect the user against typical low-level cryptographic gotchas and problems.

## Scheduler handling

The major problem which a NIF library has to address is the problem of blocking Erlang schedulers. A long-running NIF messes with the scheduler in many ways, the worst of which is breaking it. To avoid this, we have to address long-running work on the NIF. The current method used is to care about the *progress* of the system rather than the *precision*. That is, we guarantee the system will always quickly progress toward a new process, even when running many cryptographic NIFs are run in a given process. However, we don't care about the precision of the progress. A cryptographic NIF may get either a free ride on the reduction budget, or be penalized more than it should be.

The current approach is to switch between blocking NIF calls and dirty scheduler use at a breakoff threshold. Currently, we use the meaurements obtained by assuming a schedule of 100μs is 1/10th of a 1ms budget. And then we set a reduction budget based on these values. 100μs is roughly set at 200 reductions. And to be on the safe side, we multiply these values by two to handle older CPUs as well too. Measurements are obtained by running:

	enacl_timing:all().
	
The current "typical modern machine" is:

	Intel Core i7-4900QM
	
When running benchmarks, we warm the CPU a bit before conducting the benchmark. Also, the script `benchmark.sh` can be used (altered to your CPU type), to disable the powersave mode of CPUs in order to obtain realistic benchmarks. Do note nothing was done to get a realistic disable of Intel's Turbo Boost functionality and this is a one-core benchmark.

I'm interested in machines for which the schedules end up being far off. That is, machines for which the current CPU schedule takes more than 250μs. This is especially interesting for virtual machines. If you are running on very slow machines, you may have to tune the reduction counts and threshold sizes to get good latency on the system.

# Testing

Every primitive has been stress-tested through the use of Erlang QuickCheck with both *positive* and *negative* testing. This has been used to check against memory leaks as well as correct invocation. Please report any error so we can extend the test cases to include a randomized test which captures the problem so we generically catch every problem in a given class of errors.

Positive and negative testing refers to Type I and Type II errors in statistical testing. This means false positives—given a *valid* input the function rejects it; as well as false negatives—given an *invalid* input the functions fails to reject that input.

The problem however, is that while we are testing the API level, we can't really test the strength of the cryptographic primitives. We can verify their correctness by trying different standard correctness tests for the primitives, verifying that the output matches the expected one given a specific input. But there is no way we can show that the cryptographic primitive has the strength we want. Thus, we opted to mostly test the API and its invocation for stability.

Also, in addition to correctness, testing the system like this makes sure we have no memory leaks as they will show themselves under the extensive QuickCheck test cases we run. It has been verified there are no leaks in the code.

