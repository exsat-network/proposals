# Adjusting Vaulta Memory Limits to Enable FUSD on EVM

## Overview
### MSIG to be Approved: 

Vaulta smart contracts impose a strict 64 KiB limit on the combined size of the initial memory segment (Stack + Data), with a default stack size of 8 KB. This constraint has become a critical bottleneck in the Vaulta-EVM environment, where Ethereum-style external calls are implemented as recursive function calls—consuming significant stack space. As EVM-compatible features and data sections grow, available stack space shrinks, reducing the maximum external call depth to only ~10 levels, which is insufficient for modern DeFi applications—especially those using layered proxy patterns common in Real World Asset (RWA) tokenization.

This issue directly blocks Fosun Group’s deployment of its RWA-backed stablecoin FUSD on the exSat/Vaulta network. Their minting logic—requiring deep Oracle interactions—triggers a stack overflow ("access violation") during execution. Since Fosun uses identical contract logic across multiple chains, refactoring solely for exSat is not feasible.

Fortunately, Antelope 3.1 already supports increasing this limit, and all nodes have long supported the feature. We propose activating the “high” configuration, which raises the initial memory cap from 64 KiB to 1 MB and relaxes related WASM runtime constraints. This profile has been battle-tested in production for years and can be deployed safely.

Adopting this change will immediately resolve the call-depth limitation, enabling complex, EVM-compatible DeFi and RWA contracts to run reliably on Vaulta. It is a strategic necessity to support flagship partnerships like Fosun’s and to remain competitive in the rapidly growing RWA ecosystem.


## Background of FUSD
FUSD is designed to serve as a bridge between traditional financial assets and the Web3 ecosystem. Unlike traditional stablecoins, FUSD offers native yield derived from its underlying assets, including high-liquidity money market funds from top-tier institutions such as BNY Mellon, ChinaAMC, and Taikang.

This structure allows holders to capture the returns of the underlying reserve assets while remaining fully liquid within the DeFi ecosystem. By using Avalanche C-Chain as its primary and native liquidity hub, FUSD will serve as a building block for decentralized lending, trading, and collateral management, providing institutional investors with a transparent and efficient alternative to legacy cash management tools.

### Key features of FUSD
Institutional Grade: Specifically designed for family offices, private equity funds, and pension funds requiring high transparency and rapid settlement.

Yield-Bearing: Provides stable returns backed by regulated, real-world financial instruments.

Minted Natively on exSat: While multi-chain by design, FUSD will debut on exSat EVM, which will host its primary liquidity and DeFi integrations.

Secure: Built to facilitate compliant interoperability across global public blockchains.

FUSD has launched on Avalanche C-chain and will launch on Solana, reference link is here:
- https://www.avax.network/about/blog/fosun-wealths-asian-yield-bearing-rwa-stablecoin-finds-its-home-on-avalanche



## Background: Memory Architecture and Constraints
### Stack Size Limit of Vaulta Contracts
Memory within a smart contract is a finite resource; consequently, Vaulta imposes strict constraints on memory layout. The maximum linear memory size is capped at $33\text{ MiB}$ (which can be allocated prior to VM execution or grown during execution). However, any memory initialized prior to VM execution (via "WASM data sections") is strictly limited to the first $64\text{ KiB}$.

To optimize resource usage, the Contract Development Toolkit (cdt) utilizes a specific memory layout starting at address 0x00:

1. STACK
2. DATA (Initialized globals, strings, etc.)
3. HEAP

The Stack is positioned at the beginning of the memory space so that a stack overflow triggers an immediate contract crash, preventing "undefined behavior" such as silent corruption of the Data section. The default Stack size is $8\text{ KB}$ and is configurable at compile time. However, under this architecture, the user is rigidly constrained to a combined total of $64\text{ KiB}$ for both STACK + DATA.

While Antelope 3.1 introduced the ability to adjust these limits, they have not yet been updated on Vaulta and remain at the default values.

### External Call Depth Limit of EVM
Historically, the $64\text{ KiB}$ limit for the initial segment (Stack + Data) was sufficient, and developers often engineered workarounds for edge cases. However, the introduction of the Vaulta-EVM system and the Exsat Network—which utilizes the same technology—has exacerbated this constraint.

Vaulta-EVM effectively runs an Ethereum Virtual Machine inside a Vaulta contract. In this architecture, nested external calls within the EVM are directly translated into recursive function calls to execute the CALL opcode. This process is inherently stack-intensive. Although Vaulta-EVM was configured with a high Stack allocation to mitigate this, the continuous addition of new features and EIPs has caused the Data section to expand. This expansion encroaches on the memory reserved for the Stack, effectively reducing the maximum external call depth to approximately 10 levels.

This 10-level limit is insufficient for modern DeFi ecosystems, where contracts are increasingly complex and often require deep proxy layers for upgradability.

## Problem: Critical Partner Blocked by Architecture
### Partner Context: Fosun Group and RWA
[Insert existing materials regarding Fosun Group and their RWA initiative here. This section should briefly outline Fosun's scale, the specific assets being tokenized, and the strategic value of the FUSD token.]

### The Bottleneck

Exsat is currently collaborating with Fosun Group to bring these compliant Real World Assets to the Exsat network, and by extension, the Vaulta network. A critical step in this partnership is the deployment of their asset-bearing token, FUSD, on the Exsat Network.

While the Vaulta-EVM successfully deployed the FUSD and supporting contracts, the token minting process failed with an "access violation" caused by a stack overflow. The minting logic requires complex Oracle interactions to value the underlying assets, pushing the call depth beyond current limits. Since Fosun is launching these RWAs across multiple networks using identical logic, requesting an extensive refactor to accommodate Exsat-specific stack limitations is not feasible.

### Strategic Importance
RWA tokenization is currently the dominant narrative in the blockchain industry. Successfully launching Fosun’s RWA on Exsat and Vaulta is a pivotal opportunity to demonstrate technical capability and secure future growth. Conversely, as DeFi complexity increases, other partners will inevitably hit this same ceiling. Resolving this memory limit immediately is vital for maintaining competitiveness and embracing the RWA era.

### Solution: Protocol Parameter Update
As noted, Antelope 3.1 allows for the adjustment of the strict $64\text{ KiB}$ limit. We propose executing the specific action defined to update this limit:
(Reference: eosio.system.cpp Line 226)
### Implementation Details
- Mechanism: We will apply the "high" configuration setting.

- Effect: The primary change increases the initial memory limit from $64\text{ KiB}$ to $1\text{ MB}$. For completeness, it is important to note that this setting also relaxes several other ancillary memory constraints within the WASM runtime. These adjustments are bundled together in the "high" profile to support more robust contract execution.

- Feasibility: This feature was activated in the node software years ago and is supported by all nodes. The "high" configuration—including the adjustments to both the stack limit and the other auxiliary memory limits—has been well-tested in production environments for years. This ensures that the upgrade can be enabled with a high degree of confidence and safety.

### Outcome
Implementing this change will allow EVM contracts to deploy with significantly larger stack configurations. This will directly resolve the call depth bottleneck, supporting the recursion required by the CALL opcode and enabling the deployment of complex, modern DeFi contracts on both the Vaulta and EVM domains.
