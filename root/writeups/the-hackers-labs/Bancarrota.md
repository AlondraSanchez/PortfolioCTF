# Overview

**Target**: [Bancarrota](https://labs.thehackerslabs.com/machines/193)  
**Difficulty:** Easy  

<details>  
<summary>⚠️ Quick summary (spoiler)</summary>  

The challenge consists of recovering an RSA private key from partial leakage of one of its prime factors.

</details>

# Analysis

The leaked data exposed several critical RSA parameters:

- `n` → RSA modulus  
- `e` → public exponent  
- `c` → ciphertext  
- `p_msb` → most significant bits of the prime factor `p`
- `unknown_bits` → number of missing least significant bits of `p`  

In RSA, the modulus is defined as:

`n = p × q`

While factoring `n` is computationally infeasible under normal conditions, partial leakage of a prime factor significantly reduces the security of the system.

In this case, only the most significant bits of `p` were known, leaving a small unknown portion in the lower bits. This reduces the key recovery problem to a bounded search space, making reconstruction of the prime factor feasible using Coppersmith’s method.

# Exploitation

The attack is based on expressing the unknown prime as:

`p = p0 + x`

where `p0` contains the known most significant bits and `x` represents the missing lower bits.

This transforms RSA factorization into a small-root problem over modular arithmetic.

The following SageMath implementation was used to recover the missing value:

```python
# Leaked RSA parameters
n = ...
e = 65537
c = ...
p_msb = ...
unknown_bits = 220

# Define the partial prime and the polynomial ring over Zmod(n)
p0 = p_msb << unknown_bits
PR.<x> = PolynomialRing(Zmod(n))
f = p0 + x

# Coppersmith's attack bound
X = 2^unknown_bits
roots = f.small_roots(X=X, beta=0.4)

if roots:
    print(f"Found root: {roots[0]}")
    p = p0 + Integer(roots[0])
    
    if n % p == 0:
        q = n // p
        print("Factorization successful!")
        
        # Calculate private key and decrypt
        phi = (p - 1) * (q - 1)
        d = inverse_mod(e, phi)
        m = pow(c, d, n)
        
        # Convert integer to bytes 
        hex_msg = hex(m)[2:]
        if len(hex_msg) % 2 != 0: hex_msg = '0' + hex_msg
        
        print(f"Value of p: {p}")
        print(f"Value of q: {q}")
        print(f"*********************************************************")
        print(f"Message: {bytes.fromhex(hex_msg).decode(errors='ignore')}")
    else:
        print("Root does not divide n.")
else:
    print("No roots found. Try adjusting beta.")
```

The `small_roots()` function applies Coppersmith’s method internally to efficiently recover the bounded unknown value `x`.

The `beta` parameter influences the lattice construction used internally by Coppersmith’s method, affecting the balance between search space size and the success probability of finding small roots.

While `beta = 0.5` is commonly used as a starting heuristic for MSB-based attacks, in this case it did not yield valid roots. Reducing it to `beta = 0.4` adjusted the lattice constraints, allowing the algorithm to successfully recover the solution.

Once the correct root was found, the full prime factor `p` was reconstructed. With both factors known, the private exponent was computed and the ciphertext was decrypted, revealing the original message.

# Conclusion

This challenge demonstrates how partial leakage of RSA key material can completely break the cryptosystem. Even limited exposure of a prime factor is sufficient to reduce factorization to a tractable bounded root problem using Coppersmith’s method.

# Real-World Impact

This scenario demonstrates a critical failure in RSA key generation security. Even partial exposure of a prime factor is sufficient to fully compromise the private key.

In a real-world banking environment, this would have severe consequences:

- Complete compromise of encrypted communications protected by the affected key pair  
- Ability for an attacker to decrypt previously captured traffic (if recorded)  
- Impersonation of legitimate services relying on the affected certificate or key  
- Potential compromise of sensitive financial or authentication data  

Since RSA security relies on the infeasibility of factorizing `n`, any reduction in entropy of its prime components directly undermines the entire cryptographic system.

# Mitigation

To prevent attacks of this nature, the following countermeasures should be implemented:

- Ensure that private key generation does not expose partial prime information at any stage of the system lifecycle  
- Use cryptographically secure key generation libraries that avoid deterministic or partially leaked parameters  
- Regularly audit cryptographic implementations to ensure no sensitive material is logged or exposed through debugging or backups  
- Rotate cryptographic keys periodically to limit the impact of potential exposure  
- Monitor for abnormal cryptographic behavior, especially in systems handling key generation or storage  

In case of confirmed leakage of any portion of a private key, the affected key pair must be considered fully compromised and replaced immediately.