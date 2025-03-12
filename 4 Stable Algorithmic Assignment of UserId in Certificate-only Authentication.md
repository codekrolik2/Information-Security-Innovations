Let’s consider a task - it is necessary to implement a secure portal, within which users would be confident that no personal information or any other identifying information will be stored and cannot be transmitted or illegally obtained/stolen by third parties or organizations. Let’s examine the problem from the perspective of the owner of such a portal.

Let’s start with the basics - first of all, users should not save any such information in their profiles or upload it to the portal in any other way, but this is a matter of user OpSec, and the portal owner is not responsible for it.

Secondly, as we mentioned, all interactions are carried out using certificates, which means that each user will have a trusted certificate. Accordingly, the process of issuing certificates/signing CSRs, however it is implemented, ideally should not leave any logs or traces. Thus, we eliminate information about how many certificates exist, what these certificates are, and what their public keys are.

Thirdly, if we do not have information about user certificates, and we intentionally do not store it anywhere, we can still authenticate based on the Root CA; however, the question arises - how to implement authorization, that is, to match the data obtained during authentication with what is stored in our system, for example, userId.

As already noted, we cannot store the correspondence between certificate-userId or public key-userId in our database, as in the event of its compromise, this information would leak and could subsequently be matched with specific users based on the presence of certificates discovered through leaks. Since we cannot store this correspondence explicitly, the idea arises of algorithmically forming such a UserId from the authentication data, which should not be stored anywhere.

Let’s list the goals we would like to achieve with this approach:

    Minimal interaction with authentication data on the protocol, minimizing the risk of leaking authentication tokens.
        In this case, tokens are user certificates and public keys.
        On the protocol, we want to use them very quickly at the connection initialization stage and then forget them forever until the next authentication.
        And we definitely do not want to persist them anywhere at the level of our portal, in any database, logs, etc.
    From the data we store on the portal, it should be impossible to directly compute the authentication data.
        That is, from the authentication data (certificate), our portal can compute UserId, but not vice versa.
        Thus, if we leak UserId, it should not be trivially possible to match it to the certificate.
    Prevention of replay attacks on authentication.
    With a known user certificate and its keys, it should not be possible to compute UserId without full access to the portal's secrets.
        From the authentication data (certificate), our portal can compute UserId, but only our portal can do this, and no other actor, not even the user themselves.

Having defined these goals, we can propose the following solution in the form of a protocol and an algorithm for computing UserId on the portal side.

    Initialize an mTLS session, as usual.
        In this process, we obtain the user’s certificate, which is checked for trustworthiness through the Root CA in a standard manner.
    Challenge the user - for example, to sign something random.
        Optionally, as something similar already happens at the mTLS level.
        We just want to explicitly mention this point, as it serves as protection against replay attacks.
    Then the user sends us a secret packet - their public key, encrypted with their private key.
        The portal can verify this secret by decrypting it with the public key, resulting in the same public key.
    If all checks pass, the server concatenates, XORs, or otherwise merges the secret packet and the portal secret.
        The SHA-256 of the result of this operation is computed.
        Even better, run it through PBKDF2 a secret number of iterations.
    The result of step 4 is your UserId.

Let’s consider how this approach achieves the above goals.

    Minimal interaction with authentication data on the protocol.
        After computing UserId, we can completely remove the certificate from memory. For authorized and secure communication, only the TLS AES key and the computed UserId are sufficient.
    From the data we store on the portal, it should be impossible to directly compute the authentication data.
        Since we do not persist public keys or certificates, including in logs, and at the initialization of each connection, they must be destroyed immediately after authorization, even from RAM, the leakage of this data from the portal's storage is impossible.
        In the event of a complete compromise of the portal and interception of all control, of course, this data could be intercepted over time from new sessions, but there is nothing to be done if the portal is fully compromised.
        In this case, however, we are not talking about



Let’s consider how this approach achieves the aforementioned goals.

    Minimal interaction with authentication data on the protocol
        After computing the UserId, we can completely remove the certificate from memory. For authorized and secure communication, only the TLS AES key and the computed UserId are sufficient.

    It should be impossible to directly compute authentication data from the data we store on the portal
        Since we do not persist public keys or certificates, including in logs, and at the initialization of each connection, they must be destroyed immediately after authorization, even from RAM, the leakage of this data from the portal's storage is impossible.
        In the event of a complete compromise of the portal and interception of all control, of course, this data could be intercepted over time from new sessions, but there is nothing to be done if the portal is fully compromised.
        In this case, however, we are not talking about something serious, like the leakage of email or phone numbers, but about certificates and public keys, which in itself represents minimal risk, and they are very easy (and even necessary) to change regularly.

    Prevention of replay attacks on authentication
        Although the user’s secret packet could theoretically be stolen, albeit with great difficulty, and then reused, steps 1 and 2 prevent the possibility of replay attacks without a complete compromise of the user’s keys.

    With a known user certificate and its keys, it should not be possible to compute UserId
        Since in step 4 we modify the secret packet, combining it with an unknown portal secret before computing the UserId, it follows that to repeat the computation of UserId, one must know the portal secret, which is known only to the portal.
