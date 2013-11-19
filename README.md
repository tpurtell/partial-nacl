# ecma-nacl: Pure JavaScript (ECMAScript) version of NaCl cryptographic library.

[NaCl](http://nacl.cr.yp.to/) is a great crypto library that is not placing a burden of crypto-math choices onto developers, providing only solid high-level functionality (box - for public-key, and secret_box - for secret key  authenticated encryption), in a let's [stop blaming users](http://cr.yp.to/talks/2012.08.08/slides.pdf) of cryptographic library (e.g. end product developers, or us) manner.
Take a look at details of NaCl's design  "[The security impact of a new cryptographic
library](http://cr.yp.to/highspeed/coolnacl-20120725.pdf)".

ecma-nacl is a re-write of most important NaCl's functionality, which is ready for production, box and secret_box. Signing code still has XXX comments, indicating that the warning in [signing in NaCl](http://nacl.cr.yp.to/sign.html) should be taken seriously.

Rewrite is based on the copy of NaCl, included in this repository.
Tests are written to correspond those in C code, to make sure that output of this library is the same as that of C's version.
Besides this, we added comparison runs between ecma-nacl, and js-nacl, which is an [Emscripten](https://github.com/kripken/emscripten)-compilation of C library.
These comparison runs can be done in both node and browsers.

Your mileage may vary, but just three weeks ago, js-nacl was running 10% faster on Chrome.
Today, with new version of Chrome, it is ecma-nacl that is faster.
Given smaller size, and much better auditability of actual js code, ecma-nacl might be more preferable than js-nacl.

## NPM Package

This library is [registered on
npmjs.org](https://npmjs.org/package/ecma-nacl). To install it:

    npm install ecma-nacl

## Browser Package

make-browserified.js will let you make a browserified module. So, make sure that you have [browserify module](http://browserify.org/) to run the script. You may also modify it to suite your particular needs.

## API for secret-key authenticated encryption

Add module into code as

    var nacl = require('ecma-nacl');

[Secret-key authenticated](http://nacl.cr.yp.to/secretbox.html) encryption is provided by secret_box, which implements XSalsa20+Poly1305, and nothing else.

    // all incoming and outgoing things are Uint8Array's;
    // to encrypt, or pack plain text bytes into cipher bytes, use
    var cipher_bytes = nacl.secret_box.pack(plain_bytes, nonce, key);

    // decryption, or opening is done by
    var result_bytes = nacl.secret_box.open(cipher_bytes, nonce, key);

Above methods pack & open bytes, exactly like NaCl does. Cipher is 16 bytes longer than plain text bytes. And these 16 bytes, filed with Poly1305 authenticating output, are placed infront of all bytes with the message.

Key here is 32 bytes long. Nonce is 24 bytes. Nonce means number-used-once, i.e. it should be unique for every encrypted segment. Sometimes, when storing things, it is convenient to have nonce packed into cipher array. For this, secret_box has formatWN object, which is used analogously:

    // encrypting, and placing nonce as first 24 bytes infront NaCl's byte output layout
    var cipher_bytes = nacl.secret_box.formatWN.pack(plain_bytes, nonce, key);

    // decryption, or opening is done by
    var result_bytes = nacl.secret_box.formatWN.open(cipher_bytes, key);

    // extraction of nonce from cipher can be done as follows
    var extracted_nonce = nacl.secret_box.formatWN.copyNonceFrom(cipher_bytes);

It is important to always use different nonce, when encrypting something new with the same key. A function is provided, to advance nonce. The 24 bytes are taken as three 32-bit integers, and are advanced by 1 (oddly) or by 2 (evenly). So, when encrypting many segments of a huge file, advance nonce oddly every time. When key is shared, and is used for communication between two parties, one party's initial nonce may be oddly advanced initial nonce, received from the second party, and all other respective nonces are advanced evenly on both sides of communication. This way, unique nonces are used for every message send.

    // nonce changed in place oddly
    nacl.advanceNonceOddly(nonce);

    // nonce changed in place evenly
    nacl.advanceNonceEvenly(nonce);

It is common, that certain code needs to be given encryption/decryption functionality, but according to [principle of least authority](https://en.wikipedia.org/wiki/Principle_of_least_privilege) such code does not necessarily need to know secret key, with which encryption is done. So, there is an encryptor for opening and packing, with inbuilt even advance of the nonce, on every new cipher that is generated.

    // encryptor can be created with, indication, if cipher should be with-nonce
    var encryptor = nacl.makeSecretBoxEncryptor(key, nextNonce, isFormatWN);

    // or with
    var encryptor = nacl.secret_box.makeEncryptor(key, nextNonce, isFormatWN);

    // packing bytes is done with
    var cipher_bytes = encryptor.pack(plain_bytes);

    // opening is done with
    var result_bytes = encryptor.open(cipher_bytes);

    // when encryptor is no longer needed, key should be properly wiped from memory
    encryptor.destroy();

## API for public-key authenticated encryption

[Public-key authenticated](http://nacl.cr.yp.to/box.html) encryption is provided by box, which implements Curve25519+XSalsa20+Poly1305, and nothing else. Given pairs of secret-public keys, corresponding shared, in Diffie–Hellman sense, key is calculated (Curve25519) and is used for data encryption with secret_box (XSalsa20+Poly1305).

Given any random secret key, we can generate corresponding public key:

    var public_key = nacl.box.generate_pubkey(secret_key);

Secret key may come from browser's crypto.getRandomValues(array), or be derived from passphrase with [js-scrypt](https://github.com/tonyg/js-scrypt), which is an emscripten-compiled [original C library](http://www.tarsnap.com/scrypt.html).

There are two ways to use box. The first way is to always do two things, calculation of DH-shared key and subsequent packing/opening, in one step.

    // Alice encrypts message for Bob
    var cipher_bytes = nacl.box.pack(msg_bytes, nonce, bob_pkey, alice_skey);

    // Bob opens the message
    var msg_bytes = nacl.box.open(cipher_bytes, nonce, alice_pkey, bob_skey);

The second way is to calculate DH-shared key once and use it for packing/opening multiple messages, with box.pack_stream and box.open_stream, which are just nicknames of described above secret_box.pack and secret_box.open. Or, we may use encryptors for the following example of two-party exchanges.

Alice's side:

    // calculate DH-shared key for encryptor
    var dhshared_key = nacl.box.calc_dhshared_key(bob_pkey, alice_skey);

    // generate nonce, browser example
    var nonce = new Uint8Array(24);
    crypto.getRandomValues(nonce);

    // make encryptor
    var encryptor = nacl.makeSecretBoxEncryptor(dhshared_key, nonce, true);

    // dhshared_key was copied into encryptor, and, since it is no longer needed,
    // it should be wiped from the memory
    nacl.TypedArraysFactory.prototype.wipe(dhshared_key);

    // pack messages to bob
    var cipher_to_send = encryptor.pack(msg_bytes);

    // open mesages from bob
    var msg_from_bob = encryptor.open(received_cipher);

Bob's side:

    // calculate DH-shared key for encryptor
    var dhshared_key = nacl.box.calc_dhshared_key(alice_pkey, bob_skey);

    // get nonce from alice's first message, advance it oddly, and
    // use for encryptor, as encryptors on both sides advance nonces evenly
    var nonce = nacl.secret_box.formatWN.copyNonceFrom(cipher1_from_alice);
    nacl.advanceNonceOddly(nonce);

    // make encryptor
    var encryptor = nacl.makeSecretBoxEncryptor(dhshared_key, nonce, true);

    // dhshared_key was copied into encryptor, and, since it is no longer needed,
    // it should be wiped from the memory
    nacl.TypedArraysFactory.prototype.wipe(dhshared_key);

    // pack messages to alice
    var cipher_to_send = encryptor.pack(msg_bytes);

    // open mesages from alice
    var msg_from_alice = encryptor.open(received_cipher);

## Random number generation

NaCl does not do it. The randombytes in the original code is a unix shim with the following rational, given in the comment, quote: "it's really stupid that there isn't a syscall for this".

So, you should obtain cryptographically strong random bytes yourself. In node, there is crypto. There is crypto in browser. IE6? IE6 must die! Stop supporting insecure crap! Respect your users, and tell them truth, that they need modern secure browser(s).

## Signing

It is still not settled into production code in NaCl. Period. When it is ready, we will be able to serve it.

[DNSCurve](http://dnscurve.org/) is questioning whether common places of signing be better served with public-key encryption.

## XSP file format

Each NaCl's cipher must be read completely, before any plain text output.
Such requirement makes reading big files awkward.
Thus, the simplest solution is to pack NaCl's binary ciphers into self-contained small segments.
Each segment must be encrypted with a different nonce.
When segments are same-size, there is a predictable mapping when random access is needed.
Such format we call XSP (XSalsa+Poly), and provide utility to pack and open segments.
Each file's first segment contains file header with encrypted file key, suggesting a policy of one key per file.
This may allow simpler sharing of files in a web-service setting, as only file key section needs to be re-encrypted, when a new user gets the file.

## License

This code is provided here under [Mozilla Public License Version 2.0](https://www.mozilla.org/MPL/2.0/).

NaCl C library is public domain code by Daniel J. Bernstein and others crypto gods and semi-gods. We thank thy wisdom of giving us developer-friendly library.
