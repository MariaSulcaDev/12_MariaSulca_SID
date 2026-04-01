<script lang="ts">
     import { ethers } from "ethers";
     import Web3 from "web3";
     import { onDestroy } from "svelte";

     type WalletProvider = {
          request: (args: {
               method: string;
               params?: unknown[];
          }) => Promise<unknown>;
          on?: (event: string, listener: (...args: unknown[]) => void) => void;
          removeListener?: (
               event: string,
               listener: (...args: unknown[]) => void,
          ) => void;
     };

     type WalletWindow = Window & {
          ethereum?: WalletProvider;
          pali?: WalletProvider;
          paliEthereum?: WalletProvider;
     };

     type WalletOption = {
          id: string;
          label: string;
     };

     let address = "";
     let balanceEthers = "";
     let balanceWeb3 = "";
     let chainId = "";
     let networkName = "";
     let nativeSymbol = "";
     let status = "Wallet no conectada";
     let isLoading = false;
     let connectedWallet = "";
     let activeProvider: WalletProvider | null = null;
     let isUtxo = false;

     const networkInfo: Record<string, { name: string; symbol: string }> = {
          "1": { name: "Ethereum Mainnet", symbol: "ETH" },
          "5": { name: "Goerli", symbol: "ETH" },
          "11155111": { name: "Sepolia", symbol: "SEP ETH" },
          "137": { name: "Polygon", symbol: "MATIC" },
          "80001": { name: "Polygon Mumbai", symbol: "MATIC" },
          "80002": { name: "Polygon Amoy", symbol: "MATIC" },
          "56": { name: "BNB Smart Chain", symbol: "BNB" },
          "97": { name: "BNB Smart Chain Testnet", symbol: "tBNB" },
          "57042": { name: "zkSYS PoB Devnet EVM", symbol: "TSYS" },
          "43114": { name: "Avalanche C-Chain", symbol: "AVAX" },
          "43113": { name: "Avalanche Fuji", symbol: "AVAX" },
          "42161": { name: "Arbitrum One", symbol: "ETH" },
          "421613": { name: "Arbitrum Goerli", symbol: "ETH" },
          "10": { name: "Optimism", symbol: "ETH" },
          "420": { name: "Optimism Goerli", symbol: "ETH" },
          "250": { name: "Fantom Opera", symbol: "FTM" },
          "4002": { name: "Fantom Testnet", symbol: "FTM" },
          "8453": { name: "Base", symbol: "ETH" },
          "84531": { name: "Base Goerli", symbol: "ETH" },
          "1101": { name: "Polygon zkEVM", symbol: "ETH" },
          "324": { name: "zkSync Era", symbol: "ETH" },
          "100": { name: "Gnosis Chain", symbol: "xDAI" },
          "25": { name: "Cronos", symbol: "CRO" },
          "570": { name: "Rollux", symbol: "SYS" },
          "57": { name: "Syscoin NEVM", symbol: "SYS" },
          "5700": { name: "Syscoin Tanenbaum", symbol: "tSYS" },
     };

     const shortAddress = (value: string) =>
          value ? `${value.slice(0, 6)}...${value.slice(-4)}` : "-";

     const getNetworkLabel = (id: string, name: string): string => {
          const info = networkInfo[id];
          if (info) return info.name;
          const normalized = name?.toLowerCase?.() ?? "";
          if (normalized && normalized !== "unknown") return name;
          return `Chain ID ${id}`;
     };

     const getNativeSymbol = (id: string): string =>
          networkInfo[id]?.symbol ?? "ETH";

     const detectUtxoNetwork = (
          addr: string,
     ): { name: string; symbol: string } => {
          if (addr.startsWith("tsys1") || addr.startsWith("tsys"))
               return { name: "Syscoin Testnet", symbol: "tSYS" };
          if (addr.startsWith("sys1") || addr.startsWith("sys"))
               return { name: "Syscoin Mainnet", symbol: "SYS" };
          if (
               addr.startsWith("bc1") ||
               addr.startsWith("1") ||
               addr.startsWith("3")
          )
               return { name: "Bitcoin", symbol: "BTC" };
          if (
               addr.startsWith("tb1") ||
               addr.startsWith("m") ||
               addr.startsWith("n") ||
               addr.startsWith("2")
          )
               return { name: "Bitcoin Testnet", symbol: "tBTC" };
          return { name: "UTXO Network", symbol: "" };
     };

     const fetchUtxoBalance = async (
          provider: WalletProvider,
     ): Promise<string> => {
          const methods = [
               "wallet_getBalance",
               "getBalance",
               "wallet_getAccount",
          ];
          for (const method of methods) {
               try {
                    const result = (await provider.request({ method })) as
                         | string
                         | number
                         | Record<string, unknown>;
                    if (
                         typeof result === "string" ||
                         typeof result === "number"
                    )
                         return String(result);
                    if (
                         result &&
                         typeof result === "object" &&
                         "balance" in result
                    )
                         return String(result.balance);
                    if (
                         result &&
                         typeof result === "object" &&
                         "total" in result
                    )
                         return String(result.total);
               } catch {
                    /* intentar siguiente método */
               }
          }
          return "N/A";
     };

     const handleChainChanged = () => {
          if (activeProvider) refreshData();
     };

     const handleAccountsChanged = (accounts: unknown) => {
          const accs = accounts as string[];
          if (!accs || accs.length === 0) {
               disconnectWallet();
          } else {
               address = accs[0];
               isUtxo = !address.startsWith("0x");
               if (activeProvider) refreshData();
          }
     };

     const registerListeners = (provider: WalletProvider) => {
          provider.on?.("chainChanged", handleChainChanged);
          provider.on?.("accountsChanged", handleAccountsChanged);
     };

     const removeListeners = (provider: WalletProvider) => {
          provider.removeListener?.("chainChanged", handleChainChanged);
          provider.removeListener?.("accountsChanged", handleAccountsChanged);
     };

     const refreshData = async () => {
          if (!activeProvider || !address) return;
          try {
               if (isUtxo) {
                    const utxoNet = detectUtxoNetwork(address);
                    networkName = utxoNet.name;
                    nativeSymbol = utxoNet.symbol;
                    const bal = await fetchUtxoBalance(activeProvider);
                    balanceEthers = bal;
                    balanceWeb3 = bal;
               } else {
                    const bp = new ethers.BrowserProvider(
                         activeProvider as ethers.Eip1193Provider,
                    );
                    const [eB, wB, net] = await Promise.all([
                         bp.getBalance(address),
                         new Web3(activeProvider as never).eth.getBalance(
                              address,
                         ),
                         bp.getNetwork(),
                    ]);
                    balanceEthers = Number(ethers.formatEther(eB)).toFixed(6);
                    balanceWeb3 = Number(
                         new Web3().utils.fromWei(wB, "ether"),
                    ).toFixed(6);
                    chainId = net.chainId.toString();
                    networkName = getNetworkLabel(chainId, net.name);
                    nativeSymbol = getNativeSymbol(chainId);
               }
          } catch {}
     };

     const connectWallet = async (wallet: WalletOption) => {
          const w = window as WalletWindow;

          try {
               isLoading = true;
               status = `Conectando con ${wallet.label}...`;
               if (activeProvider) removeListeners(activeProvider);

               let provider: WalletProvider | null = null;
               let accounts: string[] = [];

               if (wallet.id === "pali") {
                    if (w.paliEthereum) {
                         try {
                              accounts = (await w.paliEthereum.request({
                                   method: "eth_requestAccounts",
                              })) as string[];
                              if (
                                   accounts.length &&
                                   accounts[0]?.startsWith("0x")
                              ) {
                                   provider = w.paliEthereum;
                              }
                         } catch {}
                    }

                    if (!provider && w.pali) {
                         try {
                              accounts = (await w.pali.request({
                                   method: "eth_requestAccounts",
                              })) as string[];
                              if (accounts.length) provider = w.pali;
                         } catch {}
                    }
               } else {
                    if (!w.ethereum) {
                         status = `No se detectó ${wallet.label}`;
                         return;
                    }
                    accounts = (await w.ethereum.request({
                         method: "eth_requestAccounts",
                    })) as string[];
                    provider = w.ethereum;
               }

               if (!provider || !accounts.length) {
                    status = "No se obtuvo ninguna cuenta";
                    return;
               }

               address = accounts[0];
               activeProvider = provider;
               connectedWallet = wallet.label;

               if (address.startsWith("0x")) {
                    isUtxo = false;
                    const bp = new ethers.BrowserProvider(
                         provider as ethers.Eip1193Provider,
                    );
                    const [eB, wB, net] = await Promise.all([
                         bp.getBalance(address),
                         new Web3(provider as never).eth.getBalance(address),
                         bp.getNetwork(),
                    ]);
                    balanceEthers = Number(ethers.formatEther(eB)).toFixed(6);
                    balanceWeb3 = Number(
                         new Web3().utils.fromWei(wB, "ether"),
                    ).toFixed(6);
                    chainId = net.chainId.toString();
                    networkName = getNetworkLabel(chainId, net.name);
                    nativeSymbol = getNativeSymbol(chainId);
               } else {
                    isUtxo = true;
                    const utxoNet = detectUtxoNetwork(address);
                    networkName = utxoNet.name;
                    nativeSymbol = utxoNet.symbol;
                    chainId = "UTXO";
                    const bal = await fetchUtxoBalance(provider);
                    balanceEthers = bal;
                    balanceWeb3 = bal;
               }

               status = `Conectado con ${wallet.label}`;
               registerListeners(provider);
          } catch {
               status = `No fue posible conectar con ${wallet.label}`;
          } finally {
               isLoading = false;
          }
     };

     const disconnectWallet = () => {
          if (activeProvider) removeListeners(activeProvider);
          address = "";
          balanceEthers = "";
          balanceWeb3 = "";
          chainId = "";
          networkName = "";
          nativeSymbol = "";
          connectedWallet = "";
          activeProvider = null;
          isUtxo = false;
          status = "Wallet no conectada";
     };

     let availableWallets: WalletOption[] = [];
     if (typeof window !== "undefined") {
          setTimeout(() => {
               const w = window as WalletWindow;
               const wallets: WalletOption[] = [];
               if (w.pali || w.paliEthereum)
                    wallets.push({ id: "pali", label: "Pali Wallet" });
               if (w.ethereum)
                    wallets.push({ id: "metamask", label: "MetaMask" });
               availableWallets = wallets;
          }, 100);
     }

     onDestroy(() => {
          if (activeProvider) removeListeners(activeProvider);
     });

     type EvmNetworkParam = {
          chainId: string;
          chainName: string;
          nativeCurrency: { name: string; symbol: string; decimals: number };
          rpcUrls: string[];
          blockExplorerUrls?: string[];
     };

     const evmNetworkParams: Record<string, EvmNetworkParam> = {
          "1": {
               chainId: "0x1",
               chainName: "Ethereum Mainnet",
               nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
               rpcUrls: ["https://ethereum.publicnode.com"],
               blockExplorerUrls: ["https://etherscan.io"],
          },
          "11155111": {
               chainId: "0xaa36a7",
               chainName: "Sepolia",
               nativeCurrency: {
                    name: "Sepolia ETH",
                    symbol: "SEP ETH",
                    decimals: 18,
               },
               rpcUrls: ["https://rpc.sepolia.org"],
               blockExplorerUrls: ["https://sepolia.etherscan.io"],
          },
          "137": {
               chainId: "0x89",
               chainName: "Polygon",
               nativeCurrency: { name: "MATIC", symbol: "MATIC", decimals: 18 },
               rpcUrls: ["https://polygon-rpc.com"],
               blockExplorerUrls: ["https://polygonscan.com"],
          },
          "80002": {
               chainId: "0x13882",
               chainName: "Polygon Amoy",
               nativeCurrency: { name: "MATIC", symbol: "MATIC", decimals: 18 },
               rpcUrls: ["https://rpc-amoy.polygon.technology"],
               blockExplorerUrls: ["https://amoy.polygonscan.com"],
          },
          "56": {
               chainId: "0x38",
               chainName: "BNB Smart Chain",
               nativeCurrency: { name: "BNB", symbol: "BNB", decimals: 18 },
               rpcUrls: ["https://bsc-dataseed.binance.org"],
               blockExplorerUrls: ["https://bscscan.com"],
          },
          "97": {
               chainId: "0x61",
               chainName: "BNB Smart Chain Testnet",
               nativeCurrency: { name: "tBNB", symbol: "tBNB", decimals: 18 },
               rpcUrls: ["https://data-seed-prebsc-1-s1.bnbchain.org:8545"],
               blockExplorerUrls: ["https://testnet.bscscan.com"],
          },
          "43114": {
               chainId: "0xa86a",
               chainName: "Avalanche C-Chain",
               nativeCurrency: { name: "AVAX", symbol: "AVAX", decimals: 18 },
               rpcUrls: ["https://api.avax.network/ext/bc/C/rpc"],
               blockExplorerUrls: ["https://snowtrace.io"],
          },
          "42161": {
               chainId: "0xa4b1",
               chainName: "Arbitrum One",
               nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
               rpcUrls: ["https://arb1.arbitrum.io/rpc"],
               blockExplorerUrls: ["https://arbiscan.io"],
          },
          "10": {
               chainId: "0xa",
               chainName: "Optimism",
               nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
               rpcUrls: ["https://mainnet.optimism.io"],
               blockExplorerUrls: ["https://optimistic.etherscan.io"],
          },
          "8453": {
               chainId: "0x2105",
               chainName: "Base",
               nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
               rpcUrls: ["https://mainnet.base.org"],
               blockExplorerUrls: ["https://basescan.org"],
          },
          "57": {
               chainId: "0x39",
               chainName: "Syscoin NEVM",
               nativeCurrency: { name: "Syscoin", symbol: "SYS", decimals: 18 },
               rpcUrls: ["https://rpc.syscoin.org"],
               blockExplorerUrls: ["https://explorer.syscoin.org"],
          },
          "5700": {
               chainId: "0x1644",
               chainName: "Syscoin Tanenbaum",
               nativeCurrency: {
                    name: "tSyscoin",
                    symbol: "tSYS",
                    decimals: 18,
               },
               rpcUrls: ["https://rpc.tanenbaum.io"],
               blockExplorerUrls: ["https://tanenbaum.io"],
          },
          "570": {
               chainId: "0x23a",
               chainName: "Rollux",
               nativeCurrency: { name: "Syscoin", symbol: "SYS", decimals: 18 },
               rpcUrls: ["https://rpc.rollux.com"],
               blockExplorerUrls: ["https://explorer.rollux.com"],
          },
          "57042": {
               chainId: "0xdea2",
               chainName: "zkSYS PoB Devnet EVM",
               nativeCurrency: {
                    name: "tSyscoin",
                    symbol: "TSYS",
                    decimals: 18,
               },
               rpcUrls: ["https://rpc.zkdev.nevm.syscoin.org"],
          },
          "100": {
               chainId: "0x64",
               chainName: "Gnosis Chain",
               nativeCurrency: { name: "xDAI", symbol: "xDAI", decimals: 18 },
               rpcUrls: ["https://rpc.gnosischain.com"],
               blockExplorerUrls: ["https://gnosisscan.io"],
          },
          "250": {
               chainId: "0xfa",
               chainName: "Fantom Opera",
               nativeCurrency: { name: "Fantom", symbol: "FTM", decimals: 18 },
               rpcUrls: ["https://rpc.ftm.tools"],
               blockExplorerUrls: ["https://ftmscan.com"],
          },
     };

     let isSwitching = false;

     const switchNetwork = async (targetChainId: string) => {
          if (!activeProvider || isSwitching) return;
          isSwitching = true;
          const hexId = "0x" + parseInt(targetChainId, 10).toString(16);
          try {
               await activeProvider.request({
                    method: "wallet_switchEthereumChain",
                    params: [{ chainId: hexId }],
               });
          } catch (err: unknown) {
               const e = err as { code?: number };
               if (e?.code === 4902 || e?.code === -32603) {
                    const params = evmNetworkParams[targetChainId];
                    if (params) {
                         try {
                              await activeProvider.request({
                                   method: "wallet_addEthereumChain",
                                   params: [params],
                              });
                         } catch {}
                    }
               }
          } finally {
               isSwitching = false;
          }
     };
</script>

<main
     class="grid min-h-screen place-items-center bg-[radial-gradient(circle_at_10%_0%,#f6f8ff_0%,#edf1fb_35%,#e6ebf6_100%)] p-5 text-slate-900 font-[Manrope,Segoe_UI,sans-serif]"
>
     <section
          class="w-full max-w-220 rounded-[18px] border border-[#d6dce8] bg-white/90 p-5 shadow-[0_14px_42px_-28px_rgba(15,23,42,0.45)] backdrop-blur-xs md:p-8"
     >
          <header
               class="mb-5 flex flex-col gap-4 sm:flex-row sm:items-start sm:justify-between"
          >
               <div>
                    <p
                         class="mb-[0.35rem] text-[0.72rem] font-bold uppercase tracking-[0.11em] text-slate-600"
                    >
                         PeruConfia · {connectedWallet || "Multi-Wallet"}
                    </p>
                    <h1
                         class="m-0 text-[clamp(1.45rem,1.8vw,1.95rem)] leading-[1.15] text-slate-900"
                    >
                         Panel de conexión
                    </h1>
               </div>

               <div class="flex flex-col gap-2 sm:flex-row">
                    {#each availableWallets as wallet}
                         <button
                              class="w-full min-w-42.5 rounded-[10px] bg-slate-900 px-4 py-[0.72rem] text-[0.92rem] font-bold text-white transition duration-150 enabled:hover:-translate-y-px enabled:hover:bg-slate-800 enabled:hover:shadow-[0_8px_18px_-12px_rgba(15,23,42,0.8)] disabled:cursor-not-allowed disabled:opacity-[0.65] sm:w-auto"
                              on:click={() => connectWallet(wallet)}
                              disabled={isLoading}
                         >
                              {isLoading ? "Conectando..." : wallet.label}
                         </button>
                    {/each}
                    {#if availableWallets.length === 0}
                         <p class="text-[0.85rem] text-slate-500">
                              No se detectó ningún wallet. Instala MetaMask o
                              Pali Wallet.
                         </p>
                    {/if}
               </div>
          </header>

          <div
               class="mb-[1.1rem] flex items-center gap-[0.55rem] rounded-[10px] border border-slate-200 bg-slate-50 px-[0.85rem] py-[0.7rem]"
          >
               <span
                    class="h-[0.52rem] w-[0.52rem] flex-none rounded-full bg-blue-600 shadow-[0_0_0_5px_rgba(37,99,235,0.12)]"
                    aria-hidden="true"
               ></span>
               <p class="m-0 text-[0.9rem] font-semibold text-slate-700">
                    {status}
               </p>
          </div>

          {#if address && !isUtxo}
               <div
                    class="mb-[1.1rem] rounded-xl border border-slate-200 bg-slate-50 p-3"
               >
                    <p
                         class="mb-2 text-[0.72rem] font-bold uppercase tracking-widest text-slate-500"
                    >
                         Cambiar red
                    </p>
                    <div class="flex flex-wrap gap-2">
                         {#each Object.entries(evmNetworkParams) as [cid, net]}
                              <button
                                   class="rounded-lg border px-3 py-[0.4rem] text-[0.8rem] font-semibold transition duration-150
                                        {chainId === cid
                                        ? 'border-blue-500 bg-blue-600 text-white shadow-[0_0_0_3px_rgba(37,99,235,0.15)]'
                                        : 'border-slate-200 bg-white text-slate-700 hover:border-blue-400 hover:text-blue-600'}
                                        disabled:cursor-not-allowed disabled:opacity-50"
                                   on:click={() => switchNetwork(cid)}
                                   disabled={isSwitching || chainId === cid}
                              >
                                   {net.chainName}
                              </button>
                         {/each}
                    </div>
               </div>
          {/if}

          <section
               class="grid grid-cols-1 gap-[0.85rem] sm:grid-cols-2"
               aria-label="Resumen de wallet"
          >
               <article
                    class="rounded-xl border border-slate-200 bg-white p-4 sm:col-span-2"
               >
                    <h2
                         class="m-0 text-[0.78rem] font-extrabold uppercase tracking-[0.08em] text-slate-500"
                    >
                         Dirección
                    </h2>
                    <p
                         class="mt-[0.65rem] overflow-wrap-anywhere text-[0.9rem] leading-[1.45] text-slate-900 font-['JetBrains_Mono',Consolas,monospace]"
                    >
                         {address || "-"}
                    </p>
                    <small
                         class="mt-[0.45rem] block text-[0.82rem] text-slate-500"
                         >Vista corta: {shortAddress(address)}</small
                    >
               </article>

               <article class="rounded-xl border border-slate-200 bg-white p-4">
                    <h2
                         class="m-0 text-[0.78rem] font-extrabold uppercase tracking-[0.08em] text-slate-500"
                    >
                         Saldo (ethers)
                    </h2>
                    <p
                         class="mt-[0.65rem] text-[clamp(1.05rem,2.2vw,1.35rem)] font-extrabold text-slate-900"
                    >
                         {balanceEthers || "-"}
                         {nativeSymbol || ""}
                    </p>
               </article>

               <article class="rounded-xl border border-slate-200 bg-white p-4">
                    <h2
                         class="m-0 text-[0.78rem] font-extrabold uppercase tracking-[0.08em] text-slate-500"
                    >
                         Saldo (web3)
                    </h2>
                    <p
                         class="mt-[0.65rem] text-[clamp(1.05rem,2.2vw,1.35rem)] font-extrabold text-slate-900"
                    >
                         {balanceWeb3 || "-"}
                         {nativeSymbol || ""}
                    </p>
                    <small
                         class="mt-[0.45rem] block text-[0.82rem] text-slate-500"
                         >Chain ID: {chainId || "-"}</small
                    >
               </article>

               <article class="rounded-xl border border-slate-200 bg-white p-4">
                    <h2
                         class="m-0 text-[0.78rem] font-extrabold uppercase tracking-[0.08em] text-slate-500"
                    >
                         Red conectada
                    </h2>
                    <p
                         class="mt-[0.65rem] text-[clamp(1.05rem,2.2vw,1.35rem)] font-extrabold text-slate-900"
                    >
                         {networkName || "-"}
                    </p>
                    <small
                         class="mt-[0.45rem] block text-[0.82rem] text-slate-500"
                         >Identificador: {chainId || "-"}</small
                    >
               </article>
          </section>

          <footer class="mt-[1.1rem] flex justify-end">
               <button
                    class="rounded-[10px] bg-rose-100 px-4 py-[0.72rem] text-[0.92rem] font-bold text-rose-800 transition duration-150 hover:bg-rose-200"
                    on:click={disconnectWallet}
               >
                    Desconectar
               </button>
          </footer>
     </section>
</main>
