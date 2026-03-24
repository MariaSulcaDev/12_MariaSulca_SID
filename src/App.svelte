<script lang="ts">
     import { ethers } from "ethers";
     import Web3 from "web3";

     // Tipo para el proveedor del wallet con los objetos necesarios, request con metodo y parametros, on y removeListener para eventos del wallet
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

     // tipo para que detecte los objetos del wallet
     type WalletWindow = Window & {
          ethereum?: WalletProvider;
          pali?: WalletProvider;
          paliEthereum?: WalletProvider;
     };

     // Declarar constantes de datos del wallet
     let address = "";
     let balanceEthers = "";
     let balanceWeb3 = "";
     let chainId = "";
     let networkName = "";
     let status = "Wallet no conectada";
     let isLoading = false;

     // metodo de obtener el proveedor de wallet inyectado en el navegador
     const getProvider = (): WalletProvider | null => {
          const walletWindow = window as WalletWindow;
          return (
               walletWindow.pali ??
               walletWindow.paliEthereum ??
               walletWindow.ethereum ??
               null
          );
     };

     // constante de formateo de la direccion para que se muestre corta
     const shortAddress = (value: string) =>
          value ? `${value.slice(0, 6)}...${value.slice(-4)}` : "-";

     // funcion para obtener el nombre de la red a partir del chainId o del nombre proporcionado por el wallet
     const getNetworkLabel = (id: string, name: string) => {
          const normalized = name?.toLowerCase?.() ?? "";

          if (normalized && normalized !== "unknown") {
               return name;
          }

          // redes comunes con sus chain id para interpretar el nombre de la red
          const networkByChainId: Record<string, string> = {
               "1": "Ethereum Mainnet",
               "5": "Goerli",
               "11155111": "Sepolia",
               "137": "Polygon",
               "80001": "Polygon Mumbai",
               "56": "BNB Smart Chain",
               "97": "BNB Smart Chain Testnet",
               "57042": "zkSYS PoB Devnet EVM",
               "43114": "Avalanche C-Chain",
               "43113": "Avalanche Fuji",
          };

          return networkByChainId[id] ?? `Chain ID ${id}`;
     };

     // funcion para conectar con el wallet
     const connectWallet = async () => {
          const injected = getProvider();

          if (!injected) {
               status = "No se detectó Pali Wallet";
               return;
          }

          try {
               isLoading = true;
               status = "Conectando...";

               const browserProvider = new ethers.BrowserProvider(
                    injected as ethers.Eip1193Provider,
               );
               const accounts = (await injected.request({
                    method: "eth_requestAccounts",
               })) as string[];

               if (!accounts.length) {
                    status = "No se obtuvo ninguna cuenta";
                    return;
               }

               address = accounts[0];

               const [ethersBalanceRaw, web3BalanceRaw, network] =
                    await Promise.all([
                         browserProvider.getBalance(address),
                         new Web3(injected as never).eth.getBalance(address),
                         browserProvider.getNetwork(),
                    ]);

               // aca se obtiene la cantidad de ethers que se tiene en este caso TSYS
               balanceEthers = Number(
                    ethers.formatEther(ethersBalanceRaw),
               ).toFixed(6);
               balanceWeb3 = Number(
                    new Web3().utils.fromWei(web3BalanceRaw, "ether"),
               ).toFixed(6);
               chainId = network.chainId.toString();
               networkName = getNetworkLabel(chainId, network.name);
               status = "Sesión iniciada correctamente";
          } catch {
               status = "No fue posible conectar con Pali Wallet";
          } finally {
               isLoading = false;
          }
     };
     // function para desconectar el wallet y limpiar los datos mostrados
     const disconnectWallet = () => {
          address = "";
          balanceEthers = "";
          balanceWeb3 = "";
          chainId = "";
          networkName = "";
          status = "Wallet no conectada";
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
                         PeruConfia · Pali Wallet
                    </p>
                    <h1
                         class="m-0 text-[clamp(1.45rem,1.8vw,1.95rem)] leading-[1.15] text-slate-900"
                    >
                         Panel de conexión
                    </h1>
               </div>

               <button
                    class="w-full min-w-42.5 rounded-[10px] bg-slate-900 px-4 py-[0.72rem] text-[0.92rem] font-bold text-white transition duration-150 enabled:hover:-translate-y-px enabled:hover:bg-slate-800 enabled:hover:shadow-[0_8px_18px_-12px_rgba(15,23,42,0.8)] disabled:cursor-not-allowed disabled:opacity-[0.65] sm:w-auto"
                    on:click={connectWallet}
                    disabled={isLoading}
               >
                    {isLoading ? "Conectando..." : "Conectar Wallet"}
               </button>
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
                         {balanceEthers || "-"} TSYS
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
                         {balanceWeb3 || "-"} TSYS
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
