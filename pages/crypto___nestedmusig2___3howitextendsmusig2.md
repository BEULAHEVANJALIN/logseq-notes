#### How It Extends the MuSig2 Concept
	- MuSig2 already solves two critical problems in multi-signature schemes:
		- **Rogue-key attacks**
			- The rogue-key attack exploits naive key aggregation. When parties $$P_1$$ and $$P_2$$ create a joint key, if $$P_1$$ announces public key $$X_1$$ first, a malicious $$P_2$$ can compute $$X_2 = X_{malicious} - X_1$$, where $$X_{malicious}$$ is fully controlled by $$P_2$$. The resulting aggregate key $$X_{agg} = X_1 + X_2 = X_{malicious}$$ enables $$P_2$$ to sign unilaterally. MuSig2 prevents this through coefficient-based aggregation. The complete list of public keys is hashed to derive per-key coefficients: $$a_i = H(\text{all keys} \| X_i)$$. The aggregate becomes $$X_{agg} = \sum_i a_i X_i$$. Since each coefficient depends on the complete key set, adaptive key selection becomes computationally infeasible without hash collisions.
		- **Adaptive signing attacks**
			- The second vulnerability involves nonce manipulation in the signing phase. Standard Schnorr signatures require each signer to generate a random nonce $$r$$, compute $$R = r \cdot G$$, and produce partial signature $$s = r + c \cdot x$$, where $$c$$ is the challenge and $$x$$ is the secret key. A malicious party observing other participants' $$R$$ values before committing their own can select $$R_{mal}$$ to cancel contributions and forge signatures. MuSig2 addresses this through two-round commitments with multiple nonces. Each signer generates nonce pairs $$(r_1, r_2)$$ and publishes cryptographic commitments before revealing actual values. After all commitments are published, nonces are revealed and a binding factor $$b = H(X_{agg}, R_1, R_2, m)$$ is computed. The aggregate nonce becomes $$R = R_1 + b \cdot R_2$$. Since $$b$$ remains unknown until after commitment, adaptive nonce selection is prevented.
	- Nested MuSig2 applies both mechanisms: **key coefficients** and **nonce binding**, recursively at each level of a tree structure.
		- Consider an organization where a Board comprises three directors, each with individual signing keys. Standard MuSig2 aggregates these into $$X_{board}$$. If final authorization requires both Board and CFO approval, $$X_{board}$$ and $$X_{CFO}$$ serve as inputs to root-level aggregation, producing $$X_{root}$$.
		- Each internal node executes a complete MuSig2 aggregation. Children may be individual keys or aggregate keys from lower levels (the distinction is irrelevant to the protocol). The node hashes its child list, computes coefficients, and aggregates accordingly. For nonces, each child contributes nonce pairs. The parent computes local binding factors, aggregates child nonces, and passes the resulting aggregate upward.
	- **Protocol**
		- The protocol executes in three phases.
			- Phase 1: Nonce Commitment Propagation
				- Nonce commitments propagate upward through the tree. Leaf nodes generate nonce pairs $$(r_1, r_2)$$ and compute corresponding elliptic curve points $$(R_1, R_2)$$. Internal nodes collect children's nonce pairs, aggregate using local binding factors, and transmit aggregates to their parents. At the root, a single aggregate nonce pair $$(R_1^{root}, R_2^{root})$$ emerges.
			- Phase 2: Challenge Computation
				- A global challenge is computed at the root following standard Schnorr conventions:
				  $$
				  c = H(X_{root}, R_{root}, m)
				  $$
				  where $$R_{root} = R_1^{root} + b^{root} \cdot R_2^{root}$$ and $$b^{root}$$ represents the root's binding factor. This challenge value is distributed to all signers.
			- Phase 3: Partial Signature Propagation
				- Partial signatures propagate from leaves toward the root. Each leaf signer computes a partial signature incorporating the product of all coefficients and binding factors along its path to the root.
		- Example
			- Consider a organisation with two teams requiring joint approval:
				- Team A: Alice and Bob (two members)
				- Team B: Charlie (single member)
				- Root: Requires both Team A and Team B
			- Level 1: Team A Aggregation
				- Team A aggregates Alice's key $$X_A$$ and Bob's key $$X_B$$. The node computes coefficients from the hash of both keys:
				  $$
				  a_A^{team} = H(\text{``Team A"}, X_A, X_B \| X_A)
				  $$
				  $$
				  a_B^{team} = H(\text{``Team A"}, X_A, X_B \| X_B)
				  $$
				  The Team A aggregate key becomes:
				  $$
				  X_{team\_A} = a_A^{team} \cdot X_A + a_B^{team} \cdot X_B
				  $$
			- Level 2: Root Aggregation
				- The root aggregates Team A's key with Charlie's key $$X_C$$ (representing Team B). Root coefficients:
				  $$
				  a_{teamA}^{root} = H(\text{``Root"}, X_{team\_A}, X_C \| X_{team\_A})
				  $$
				  $$
				  a_C^{root} = H(\text{``Root"}, X_{team\_A}, X_C \| X_C)
				  $$
				  The root aggregate key:
				  $$
				  X_{root} = a_{teamA}^{root} \cdot X_{team\_A} + a_C^{root} \cdot X_C
				  $$
			- Alice's Path Accumulation
				- Substituting $$X_{team\_A}$$ into the root equation:
				  $$
				  X_{root} = a_{teamA}^{root} \cdot (a_A^{team} \cdot X_A + a_B^{team} \cdot X_B) + a_C^{root} \cdot X_C
				  $$
				  Alice's public key appears with coefficient:
				  $$
				  a_{teamA}^{root} \cdot a_A^{team}
				  $$
				  When signing with challenge $$c$$, Alice computes her partial signature as:
				  $$
				  s_A = r_A^{path} + c \cdot (a_{teamA}^{root} \cdot a_A^{team} \cdot x_A)
				  $$
				  where $$r_A^{path}$$ is her nonce contribution scaled by binding factors along her path: first at Team A level, then at root level.
			- Charlie's Path Accumulation
				- Charlie participates only at the root level, so his contribution uses only the root coefficient:
				  $$
				  s_C = r_C + c \cdot (a_C^{root} \cdot x_C)
				  $$
				  His path contains only one coefficient because he sits directly at depth 1.
		- Key Observation
			- The path length determines the number of coefficients in the product. Alice traverses two levels (Team A $$\rightarrow$$ Root), yielding a product of two coefficients. Charlie traverses one level (Root only), yielding a single coefficient. This demonstrates how nested structure naturally encodes organisational hierarchy into the signature algebra without external metadata.
	- #### Security Properties
		- The security reduction inherits guarantees from MuSig2. At each level, coefficient-based aggregation prevents rogue-key attacks within that node. Binding factors prevent nonce manipulation at each layer. Since hash computations at each level depend on inputs including child aggregates, forging signatures requires breaking at least one aggregation layer.
		- Producing a valid root signature without knowledge of the requisite secret keys requires generating valid partial signatures for some path through the tree. These partial signatures are bound by cumulative products of hash-derived factors. Forgery necessitates either solving the discrete logarithm problem or finding hash collisions in coefficient or binding computations.
	- #### External Indistinguishability
		- The final output is a standard Schnorr signature verifying against $$X_{root}$$. No information about tree structure, hierarchy depth, or participant count at any level is revealed. External observers cannot distinguish nested signatures from single-signer signatures. However, the signature algebraically encodes commitments to every node's aggregation structure through cumulative products, cryptographically binding every ancestor's policy into each leaf's contribution.
		- This approach eliminates the need for external scripts or layered transactions to express hierarchical policies. The tree structure embeds directly into signature algebra, preserving compactness, privacy, and provable security at every level.