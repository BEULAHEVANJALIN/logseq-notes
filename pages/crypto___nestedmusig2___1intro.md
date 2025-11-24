#### What is Nested MuSig2?
	- At its core, Nested MuSig2 is an extension of the popular MuSig2 multi-signature protocol. Both protocols allow a group of individuals, each with a secret key, to collaborate and create a single, compact Schnorr signature that can be verified using a shared aggregated public key.
	- However, Nested MuSig2 introduces the important capability of hierarchy. It allows a signature to be created not only from a flat list of keys, but also from a hierarchical structure, where an aggregated "group key" can itself be one of the signers in a higher-level aggregation. This results in a final signature that cryptographically represents an entire chain of authority.
	-
	- {{renderer :mermaid_691d6d90-0a0e-4351-848c-f1db4124e18d, 3}}
	  collapsed:: true
		- ```mermaid
		  %%{init: {
		    "themeVariables": {
		      "lineColor": "#ffffff",
		      "arrowheadColor": "#ffffff"
		    }
		  }}%%
		  flowchart TD
		      A1["A1 (leaf)"] --> A["Internal A = MuSig2(A1, A2)"]
		      A2["A2 (leaf)"] --> A
		      B1["B1 (leaf)"] --> B["Internal B = MuSig2(B1, B2)"]
		      B2["B2 (leaf)"] --> B
		      C1["C1 (leaf)"] --> C["Internal C = MuSig2(C1, C2)"]
		      C2["C2 (leaf)"] --> C
		      A --> ROOT["ROOT = MuSig2(A, B, C)"]
		      B --> ROOT
		      C --> ROOT
		  ```
-
- #### Why Nesting Is Necessary
	- Traditional multi-signature schemes use flat structures where all signers are at the same level. This works fine for simple cases, but it fails to capture how real organisations actually make decisions. Most corporate, regulatory, and governance structures are inherently hierarchical. Decisions flow through layers of authority, each with its own internal signing requirements.
	- Nesting solves this by allowing aggregate keys to combine at multiple levels. An executive committee might produce one aggregate key, a compliance team another, and both are needed for final authorization. Each team maintains its own internal multi-signature policy, but from the outside, they appear as single entities that must jointly approve the transaction.
	  Without nesting, we're forced to resort to poor solutions:
		- Flatten the hierarchy: Combine all signers into one massive multi-signature set, exposing organisational structure and making membership changes painful.
		- Use script branching: Encode multiple keys and conditions directly in on-chain scripts, creating bigger, more costly transactions that leak our entire policy structure when spent.
		- Layer transactions: Chain separate transactions and smart contracts together, sacrificing efficiency and introducing more surface attacks.
	- None of these approaches give us the clean separation of concerns that hierarchical organisations naturally require.
	- Properly designed nested schemes preserve compactness (one signature, regardless of internal complexity), flexibility (teams can modify their internal policies independently), and verifiability (the entire authorization chain is cryptographically guaranteed).