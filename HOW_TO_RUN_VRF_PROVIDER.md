# How to run the VRF provider service

In Band's ecosystem, the VRF provider service is an external API that provides the randomness. The service has to hold a secret key to produce verifiable randomness. Other than that is a single endpoint with VRF logic which Band has already provided the complete implementation.

Although there are many ways to set up the VRF API, this document will guide anyone who wants to set up the VRF provider API using the Google cloud function.

The first step is creating a cloud function by clicking the `[+] CREATE FUNCTION` button.
- ![img1](https://user-images.githubusercontent.com/12705423/192422871-6cec0f0d-ec89-4766-acbb-39f9439ee95f.png)

The next step is to fill in the Google Cloud configuration, such as function name and region. An important parameter here is the **secret key(32 bytes)**. A secret key, also known as a private key, is a cryptography variable used with an algorithm to encrypt and decrypt data. Secret keys should only be shared with the key's generator or the authorized parties. So, the secret should be generated randomly and secretly, making the key very hard to guess. We suggest that the owner of the secret key should use their method to create the key and avoid using the wildly use tool to avoid the predictability of the key.
- ![img2](https://user-images.githubusercontent.com/12705423/192370520-83b06ee0-63da-4a7b-9903-9bb49121629b.png)

After setting the configuration, click Next, and you will arrive at the Code page. The next step is to set the language to `Python3.8`, set the endpoint to `vrf`, and then create three files in the directory, as shown in the figure below.
- ![img3](https://user-images.githubusercontent.com/12705423/192373728-ac55df93-74ab-4aab-ae1a-84bbb3adacb2.png)

For each file, copy and paste the code below one by one.

- main.py 
    ```python3
    import os
    import time
    from flask import jsonify

    from vrf import get_public_key, ecvrf_prove, ecvrf_proof_to_hash

    SECRET_KEY = bytes.fromhex(os.getenv("SECRET_KEY"))
    if len(SECRET_KEY) != 32:
        raise ValueError("Missing 32-byte HEX formatted SECRET_KEY env")

    def vrf(request):
        data = request.get_json()
        available_time = int(data["timestamp"])
        seed = data["seed"]
        if available_time > int(time.time()):
            return jsonify({"error": "Too soon to reveal the random value"}), 400
        alpha_string = "{}:{}".format(seed, available_time).encode()
        pi_ok, pi_string = ecvrf_prove(SECRET_KEY, alpha_string)
        if pi_ok != "VALID":
            return jsonify({"error": "Error generating VRF proof"}), 500
        beta_ok, beta_string = ecvrf_proof_to_hash(pi_string)
        if beta_ok != "VALID":
            return jsonify({"error": "Error generating VRF hash"}), 500
        return jsonify({"proof": pi_string.hex(), "hash": beta_string.hex()})
    ```
- requirements.txt
    ```shell
    appdirs==1.4.4
    attrs==19.3.0
    black==19.10b0
    click==7.1.2
    Flask==1.1.2
    gevent==20.6.2
    greenlet==0.4.16
    gunicorn==20.0.4
    itsdangerous==1.1.0
    Jinja2==2.11.2
    MarkupSafe==1.1.1
    pathspec==0.8.0
    regex==2020.6.8
    toml==0.10.1
    typed-ast==1.4.1
    Werkzeug==1.0.1
    zope.event==4.4
    zope.interface==5.1.0
    ```
    
- vrf.py
    ```python3
    # MIT License

    # Copyright (c) 2020 Eric Schorn, NCC Group Plc

    # Permission is hereby granted, free of charge, to any person obtaining a copy
    # of this software and associated documentation files (the "Software"), to deal
    # in the Software without restriction, including without limitation the rights
    # to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    # copies of the Software, and to permit persons to whom the Software is
    # furnished to do so, subject to the following conditions:

    # The above copyright notice and this permission notice shall be included in all
    # copies or substantial portions of the Software.

    # THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    # IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    # FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    # AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    # LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    # OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    # SOFTWARE.

    # Copyright (C) 2020 Eric Schorn, NCC Group Plc; Provided under the MIT license

    # This code follows the (IETF) IRTF CFRG Verifiable Random Functions (VRFs) spec *very* closely.
    # Please refer to https://tools.ietf.org/pdf/draft-irtf-cfrg-vrf-06.pdf


    import hashlib
    import sys

    if sys.version_info[0] != 3 or sys.version_info[1] < 7:
        print("Requires Python v3.7+")
        sys.exit()


    # Public API

    # Section 5.1. ECVRF Proving
    def ecvrf_prove(sk, alpha_string):
        """
        Input:
            sk - VRF private key (32 bytes)
            alpha_string - input alpha, an octet string
        Output:
            ("VALID", pi_string) - where pi_string is the VRF proof, octet string of length ptLen+n+qLen
            (80) bytes, or ("INVALID", []) upon failure
        """
        # 1. Use sk to derive the VRF secret scalar x and the VRF public key y = x*B
        secret_scalar_x = _get_secret_scalar(sk)
        public_key_y = get_public_key(sk)

        # 2. H = ECVRF_hash_to_curve(y, alpha_string)
        h = _ecvrf_hash_to_curve_elligator2_25519(public_key_y, alpha_string)
        if h == "INVALID":
            return "INVALID", []

        # 3. h_string = point_to_string(H)
        h_string = _decode_point(h)
        if h_string == "INVALID":
            return "INVALID", []

        # 4. Gamma = x*H
        gamma = _scalar_multiply(p=h_string, e=secret_scalar_x)

        # 5. k = ECVRF_nonce_generation(sk, h_string)
        k = _ecvrf_nonce_generation_rfc8032(sk, h)

        # 6. c = ECVRF_hash_points(H, Gamma, k*B, k*H)
        k_b = _scalar_multiply(p=BASE, e=k)
        k_h = _scalar_multiply(p=h_string, e=k)
        c = _ecvrf_hash_points(h_string, gamma, k_b, k_h)

        # 7. s = (k + c*x) mod q
        s = (k + c * secret_scalar_x) % ORDER

        # 8. pi_string = point_to_string(Gamma) || int_to_string(c, n) || int_to_string(s, qLen)
        pi_string = _encode_point(gamma) + int.to_bytes(c, 16, 'little') + int.to_bytes(s, 32, 'little')

        if 'test_dict' in globals():
            _assert_and_sample(['secret_scalar_x', 'public_key_y', 'h', 'gamma', 'k_b', 'k_h', 'pi_string'],
                               [secret_scalar_x.to_bytes(32, 'little'), public_key_y, h, _encode_point(gamma),
                                _encode_point(k_b), _encode_point(k_h), pi_string])

        # 9. Output pi_string
        return "VALID", pi_string


    # Section 5.2. ECVRF Proof To Hash
    def ecvrf_proof_to_hash(pi_string):
        """
        Input:
            pi_string - VRF proof, octet string of length ptLen+n+qLen (80) bytes
        Output:
            ("VALID", beta_string) where beta_string is the VRF hash output, octet string
            of length hLen (64) bytes, or ("INVALID", []) upon failure
        """
        # 1. D = ECVRF_decode_proof(pi_string)
        d = _ecvrf_decode_proof(pi_string)

        # 2. If D is "INVALID", output "INVALID" and stop
        if d == "INVALID":
            return "INVALID", []

        # 3. (Gamma, c, s) = D
        gamma, c, s = d

        # 4. three_string = 0x03 = int_to_string(3, 1), a single octet with value 3
        three_string = bytes([0x03])
        zero_string = bytes([0x00])
        # 5. beta_string = Hash(suite_string || three_string || point_to_string(cofactor * Gamma))
        cofactor_gamma = _scalar_multiply(p=gamma, e=COFACTOR)  # Curve cofactor
        beta_string = _hash(SUITE_STRING + three_string + _encode_point(cofactor_gamma) + zero_string)

        if 'test_dict' in globals():
            _assert_and_sample(['beta_string'], [beta_string])

        # 6. Output beta_string
        return "VALID", beta_string


    # Section 5.3. ECVRF Verifying
    def ecvrf_verify(y, pi_string, alpha_string):
        """
        Input:
            y - public key, an EC point as bytes
            pi_string - VRF proof, octet string of length ptLen+n+qLen (80) bytes
            alpha_string - VRF input, octet string
        Output:
            ("VALID", beta_string), where beta_string is the VRF hash output, octet string
            of length hLen (64) bytes; or ("INVALID", []) upon failure
        """
        # Note that the API caller is expected to verify that the returned beta_string is the
        # expected one and this has a strong potential for mistakes/oversights (such as checking
        # for "VALID" but not the actual value). Production code would be better served by
        # passing in the expected beta_string and getting a simpler pass/fail in response.

        # 1. D = ECVRF_decode_proof(pi_string)
        d = _ecvrf_decode_proof(pi_string)

        # 2. If D is "INVALID", output "INVALID" and stop
        if d == "INVALID":
            return "INVALID", []

        # 3. (Gamma, c, s) = D
        gamma, c, s = d

        # 4. H = ECVRF_hash_to_curve(y, alpha_string)
        h = _ecvrf_hash_to_curve_elligator2_25519(y, alpha_string)
        if h == "INVALID":
            return "INVALID", []

        # 5. U = s*B - c*y
        y_point = _decode_point(y)
        h_point = _decode_point(h)
        if y_point == "INVALID" or h_point == "INVALID":
            return "INVALID", []
        s_b = _scalar_multiply(p=BASE, e=s)
        c_y = _scalar_multiply(p=y_point, e=c)
        nc_y = [PRIME - c_y[0], c_y[1]]
        u = _edwards_add(s_b, nc_y)

        # 6. V = s*H - c*Gamma
        s_h = _scalar_multiply(p=h_point, e=s)
        c_g = _scalar_multiply(p=gamma, e=c)
        nc_g = [PRIME - c_g[0], c_g[1]]
        v = _edwards_add(nc_g, s_h)

        # 7. c’ = ECVRF_hash_points(H, Gamma, U, V)
        cp = _ecvrf_hash_points(h_point, gamma, u, v)

        if 'test_dict' in globals():
            _assert_and_sample(['h', 'u', 'v'], [h, _encode_point(u), _encode_point(v)])

        # 8. If c and c’ are equal, output ("VALID", ECVRF_proof_to_hash(pi_string)); else output "INVALID"
        if c == cp:
            return ecvrf_proof_to_hash(pi_string)  # Includes logic for VALID/INVALID
        else:
            return "INVALID", []


    def get_public_key(sk):
        """Calculate and return the public_key as an encoded point string (bytes)
        """
        secret_int = _get_secret_scalar(sk)
        public_point = _scalar_multiply(p=BASE, e=secret_int)
        public_string = _encode_point(public_point)
        return public_string

    def _i2osp(x, xLen):
            if x >= 256**xLen:
                raise ValueError("integer too large")
            digits = []

            while x:
                digits.append(int(x % 256))
                x //= 256
            for i in range(xLen - len(digits)):
                digits.append(0)
            return bytes(digits[::-1])

    def _os2ip(X):
            xLen = len(X)
            X = X[::-1]
            x = 0
            for i in range(xLen):
                x += X[i] * 256**i
            return x

    def ecvrf_validate_key(pk_string):
        """
        Input:
            PK_string - public key, an octet string
        Output:
            "INVALID", or
            Y - public key, an EC point
        """

        # 1. Y = string_to_point(PK_string)
        y = _decode_point(pk_string)

        # 2. If Y is "INVALID", output "INVALID" and stop
        if y == 'INVALID':
            return 'INVALID'

        # 3. If cofactor*Y is the identity element of the elliptic curve group, output "INVALID" and stop
        if _scalar_multiply(y, COFACTOR) == [0,1]:
            return 'INVALID'

        # 4. Output Y
        return y

    # Internal functions
    def _expand_message_xmd(msg, dst, len_in_bytes):
        dst_prime = dst + _i2osp(len(dst), 1)
        # input block size of SHA-512 is 128
        Z_pad = _i2osp(0, 128)
        l_i_b_str = _i2osp(len_in_bytes, 2)
        msg_prime = Z_pad + msg + l_i_b_str + _i2osp(0, 1) + dst_prime
        b_0 = _hash(msg_prime)
        b_1 = _hash(b_0 + _i2osp(1, 1) + dst_prime)
        return b_1

    def _hash_to_field(msg, count):
        m = 1
        l = 48
        len_in_bytes = count * m * l
        uniform_bytes = _expand_message_xmd(msg, DST, len_in_bytes)
        tv = uniform_bytes[:48]
        return _os2ip(tv) % PRIME

    # Section 5.4.1.2. ECVRF_hash_to_curve_elligator2_25519
    def _ecvrf_hash_to_curve_elligator2_25519(y, alpha_string):
        """
        Input:
            alpha_string - value to be hashed, an octet string
            y - public key, an EC point as bytes
        Output:
            H - hashed value, a finite EC point in G, or INVALID upon failure
        Fixed options:
            p = 2^255-19, the size of the finite field F, a prime, for edwards25519 and curve25519 curves
            A = 486662, Montgomery curve constant for curve25519
            cofactor = 8, the cofactor for edwards25519 and curve25519 curves
        """
        u = _hash_to_field(y + alpha_string, 1)

        tv1 = u * u
        tv1 = (2 * tv1) % PRIME
        if tv1 == -1:
            tv1 = 0

        x1 = (tv1 + 1) % PRIME
        x1 = _inverse(x1)
        x1 = (-A * x1) % PRIME
        gx1 = (x1 + A) % PRIME
        gx1 = (gx1 * x1) % PRIME
        gx1 = (gx1 + 1) % PRIME
        gx1 = (gx1 * x1) % PRIME
        x2 = (-x1 - A) % PRIME
        gx2 = (tv1 * gx1) % PRIME

        e2 = pow(gx1, (PRIME - 1) // 2, PRIME) in [0, 1]
        if e2:
            x = x1
            gx = gx1
        else:
            x = x2
            gx = gx2

        edwards_y = (x - 1) * _inverse(x + 1) % PRIME

        h_string = int.to_bytes(edwards_y, 32, 'little')
        h_prelim = _decode_point(h_string)
        if h_prelim == "INVALID":
            return "INVALID"

        # h_prelim now has the correct y coordinate; x coordinate may have the wrong sign
        # To find if the sign is correct, convert the x-coordinate to Montomery and check
        y_coordinate = SQRT_MINUS_A_PLUS_2 * x * _inverse(h_prelim[0]) % PRIME

        # Sanity check: v should be the square root of gx
        if (y_coordinate * y_coordinate % PRIME != gx):
            return "INVALID"

        # Check if the sign needs to be changed
        e3 = y_coordinate & 1
        if e2 ^ e3:
            h_prelim[0] = -h_prelim[0] % PRIME

        # clear cofactor
        h = _scalar_multiply(p=h_prelim, e=COFACTOR)  # Curve cofactor

        # Output H
        h_point = _encode_point(h)

        return h_point


    # 5.4.2.2. ECVRF Nonce Generation From RFC 8032
    def _ecvrf_nonce_generation_rfc8032(sk, h_string):
        """
        Input:
            sk - an ECVRF secret key as bytes
            h_string - an octet string
        Output:
            k - an integer between 0 and q-1
        """
        # 1. hashed_sk_string = Hash (sk)
        hashed_sk_string = _hash(sk)

        # 2. truncated_hashed_sk_string = hashed_sk_string[32]...hashed_sk_string[63]
        truncated_hashed_sk_string = hashed_sk_string[32:]

        # 3. k_string = Hash(truncated_hashed_sk_string || h_string)
        k_string = _hash(truncated_hashed_sk_string + h_string)

        # 4. k = string_to_int(k_string) mod q
        k = int.from_bytes(k_string, 'little') % ORDER

        if 'test_dict' in globals():
            _assert_and_sample(['k'], [k_string])

        return k


    # Section 5.4.3. ECVRF Hash Points
    def _ecvrf_hash_points(p1, p2, p3, p4):
        """
        Input:
            P1...PM - EC points in G
        Output:
            c - hash value, integer between 0 and 2^(8n)-1
        """
        # 1. two_string = 0x02 = int_to_string(2, 1), a single octet with value 2
        two_string = bytes([0x02])

        # 2. Initialize str = suite_string || two_string
        string = SUITE_STRING + two_string

        # 3. for PJ in [P1, P2, ... PM]:
        #        str = str || point_to_string(PJ)
        string = string + _encode_point(p1) + _encode_point(p2) + _encode_point(p3) + _encode_point(p4) + bytes([0x00])

        # 4. c_string = Hash(str)
        c_string = _hash(string)

        # 5. truncated_c_string = c_string[0]...c_string[n-1]
        truncated_c_string = c_string[0:16]

        # 6. c = string_to_int(truncated_c_string)
        c = int.from_bytes(truncated_c_string, 'little')

        # 7. Output c
        return c


    # Section 5.4.4. ECVRF Decode Proof
    def _ecvrf_decode_proof(pi_string):
        """
        Input:
            pi_string - VRF proof, octet string (ptLen+cLen+qLen octets)
        Output:
            "INVALID", or Gamma - EC point
            c - integer between 0 and 2^(8*cLen)-1
            s - integer between 0 and q-1
        """
        if len(pi_string) != 80:  # ptLen+n+qLen octets = 32+16+32 = 80
            return "INVALID"

        # 1. let gamma_string = pi_string[0]...p_string[ptLen-1]
        gamma_string = pi_string[0:32]

        # 2. let c_string = pi_string[ptLen]...pi_string[ptLen+n-1]
        c_string = pi_string[32:48]

        # 3. let s_string =pi_string[ptLen+n]...pi_string[ptLen+n+qLen-1]
        s_string = pi_string[48:]

        # 4. Gamma = string_to_point(gamma_string)
        gamma = _decode_point(gamma_string)

        # 5. if Gamma = "INVALID" output "INVALID" and stop.
        if gamma == "INVALID":
            return "INVALID"

        # 6. c = string_to_int(c_string)
        c = int.from_bytes(c_string, 'little')

        # 7. s = string_to_int(s_string)
        s = int.from_bytes(s_string, 'little')

        # 8. if s >= q output "INVALID" and stop
        if s >= ORDER:
            return "INVALID"

        # 9. Output Gamma, c, and s
        return gamma, c, s


    def _assert_and_sample(keys, actuals):
        """
        Input:
            key - key for assert values, basename (+ '_sample') for sampled values.
        Output:
            None; asserts actuals then and assigns into global test_dict
        If key exists, assert dict expected value against provided actual value.
        Sample actual value and store into test_dict under key + '_sample'.
        """
        # noinspection PyGlobalUndefined
        global test_dict
        for key, actual in zip(keys, actuals):
            if key in test_dict and actual:
                assert actual == test_dict[key], "{}  actual:{} != expected:{}".format(key, actual.hex(), test_dict[key].hex())
            test_dict[key + '_sample'] = actual


    # Much of the following code has been adapted from ed25519.py at https://ed25519.cr.yp.to/software.html retrieved 27 Dec 2019
    # While it is gloriously inefficient, it provides an excellent demonstration of the underlying math. For example, production
    # code would likely avoid inversion via Fermat's little theorem as it is extremely expensive with a cost of ~300 field multiplies.

    def _edwards_add(p, q):
        """Edwards curve point addition"""
        x1 = p[0]
        y1 = p[1]
        x2 = q[0]
        y2 = q[1]
        denom = D * x1 * x2 * y1 * y2
        x3 = (x1 * y2 + x2 * y1) * _inverse(1 + denom)
        y3 = (y1 * y2 + x1 * x2) * _inverse(1 - denom)
        return [x3 % PRIME, y3 % PRIME]


    def _encode_point(p):
        """Encode point to string containing LSB OF X and 254 bits of y"""
        return ((p[1] & ((1 << 255) - 1)) + ((p[0] & 1) << 255)).to_bytes(32, 'little')


    def _decode_point(s):
        """Decode string containing LSB of X and 254 bits of y into point. Checks on-curve. May return \"INVALID\""""
        if int.from_bytes(s[:-1] + bytes([s[-1] & 127]),'little') >= PRIME:
            return 'INVALID'
        y = int.from_bytes(s, 'little') & ((1 << 255) - 1)
        x = _x_recover(y)
        if x & 1 != _get_bit(s, BITS - 1):
            x = PRIME - x
        p = [x, y]
        if not _is_on_curve(p):
            return "INVALID"
        return p


    def _get_bit(h, i):
        """Return specified bit from string for subsequent testing"""
        h1 = int.from_bytes(h, 'little')
        return (h1 >> i) & 0x01


    def _get_secret_scalar(sk):
        """Calculate and return the secret_scalar integer
        """
        h = bytearray(_hash(sk)[0:32])
        h[31] = int((h[31] & 0x7f) | 0x40)
        h[0] = int(h[0] & 0xf8)
        secret_int = int.from_bytes(h, 'little')
        return secret_int


    def _hash(message):
        """Return 64-byte SHA512 hash of arbitrary-length byte message"""
        return hashlib.sha512(message).digest()


    def _inverse(a):
        """Calculate inverse via Extended Euclidean Algorithm"""
        lm, hm = 1, 0
        low, high = a % PRIME, PRIME
        while low > 1:
            ratio = high//low
            nm, new = hm-lm*ratio, high-low*ratio
            lm, low, hm, high = nm, new, lm, low
        return lm % PRIME


    def _is_on_curve(p):
        """Check to confirm point is on curve; return boolean"""
        x = p[0]
        y = p[1]
        result = (-x * x + y * y - 1 - D * x * x * y * y) % PRIME
        return result == 0


    def _scalar_multiply(p, e):
        """Scalar multiplied by curve point"""
        q = [0, 1]
        for ee in bin(e)[2:]:
            q = _edwards_add(q, q)
            if ee == '1':
                q = _edwards_add(q, p)
        return q


    def _x_recover(y):
        """Recover x coordinate from y coordinate"""
        xx = (y * y - 1) * _inverse(D * y * y + 1)
        x = pow(xx, (PRIME + 3) // 8, PRIME)
        if (x * x - xx) % PRIME != 0:
            x = (x * II) % PRIME
        if x % 2 != 0:
            x = PRIME - x
        return x


    # Constants, some of which are calculated/checked at runtime using above routines
    # See https://ed25519.cr.yp.to/python/checkparams.py
    SUITE_STRING = bytes([0x04])
    DST = "ECVRF_edwards25519_XMD:SHA-512_ELL2_NU_".encode() + SUITE_STRING
    BITS = 256
    PRIME = 2 ** 255 - 19
    ORDER = 2 ** 252 + 27742317777372353535851937790883648493
    COFACTOR = 8
    TWO_INV = _inverse(2)
    II = pow(2, (PRIME - 1) // 4, PRIME)
    A = 486662
    D = -121665 * _inverse(121666)
    SQRT_MINUS_A_PLUS_2 = 6853475219497561581579357271197624642482790079785650197046958215289687604742
    BASEy = 4 * _inverse(5)
    BASEx = _x_recover(BASEy)
    BASE = [BASEx % PRIME, BASEy % PRIME]
    assert BITS >= 10
    assert 8 * len(_hash("hash input".encode("UTF-8"))) == 2 * BITS
    assert pow(2, PRIME - 1, PRIME) == 1
    assert PRIME % 4 == 1
    assert pow(2, ORDER - 1, ORDER) == 1
    assert ORDER >= 2 ** (BITS - 4)
    assert ORDER <= 2 ** (BITS - 3)
    assert pow(D, (PRIME - 1) // 2, PRIME) == PRIME - 1
    assert pow(II, 2, PRIME) == PRIME - 1
    assert _is_on_curve(BASE)
    assert _scalar_multiply(BASE, ORDER) == [0, 1]
    ```
    
After the three files are filled with the code, click the `Deploy` button and then wait for a moment for the function to be created.
- ![img4](https://user-images.githubusercontent.com/12705423/192375448-b47943e3-c604-43fb-9d0a-93443067768b.png)

If every things is find, you should see the function here. 
- ![img5](https://user-images.githubusercontent.com/12705423/192376711-abcf9d15-bc2f-4681-a2cb-90ef7aaa0ada.png)


