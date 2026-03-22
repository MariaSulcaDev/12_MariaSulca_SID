<script lang="ts">
     import { ethers } from "ethers";
     import Web3 from "web3";

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

     let address = "";
     let balanceEthers = "";
     let balanceWeb3 = "";
     let chainId = "";
     let status = "Wallet no conectada";
     let isLoading = false;

     const getProvider = (): WalletProvider | null => {
          const walletWindow = window as WalletWindow;
          return (
               walletWindow.pali ??
               walletWindow.paliEthereum ??
               walletWindow.ethereum ??
               null
          );
     };

     const shortAddress = (value: string) =>
          value ? `${value.slice(0, 6)}...${value.slice(-4)}` : "-";

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

               balanceEthers = Number(
                    ethers.formatEther(ethersBalanceRaw),
               ).toFixed(6);
               balanceWeb3 = Number(
                    new Web3().utils.fromWei(web3BalanceRaw, "ether"),
               ).toFixed(6);
               chainId = network.chainId.toString();
               status = "Sesión iniciada correctamente";
          } catch {
               status = "No fue posible conectar con Pali Wallet";
          } finally {
               isLoading = false;
          }
     };

     const disconnectWallet = () => {
          address = "";
          balanceEthers = "";
          balanceWeb3 = "";
          chainId = "";
          status = "Wallet no conectada";
     };
</script>

<main class="shell">
     <section class="panel">
          <header class="header">
               <div>
                    <p class="eyebrow">PeruConfia · Pali Wallet</p>
                    <h1>Panel de conexión</h1>
               </div>

               <button
                    class="primary"
                    on:click={connectWallet}
                    disabled={isLoading}
               >
                    {isLoading ? "Conectando..." : "Conectar Wallet"}
               </button>
          </header>

          <div class="status-row">
               <span class="dot" aria-hidden="true"></span>
               <p>{status}</p>
          </div>

          <section class="grid" aria-label="Resumen de wallet">
               <article class="card card-wide">
                    <h2>Dirección</h2>
                    <p class="mono">{address || "-"}</p>
                    <small>Vista corta: {shortAddress(address)}</small>
               </article>

               <article class="card">
                    <h2>Saldo (ethers)</h2>
                    <p class="metric">{balanceEthers || "-"} TSYS</p>
               </article>

               <article class="card">
                    <h2>Saldo (web3)</h2>
                    <p class="metric">{balanceWeb3 || "-"} TSYS</p>
                    <small>Chain ID: {chainId || "-"}</small>
               </article>
          </section>

          <footer class="actions">
               <button class="ghost" on:click={disconnectWallet}>Desconectar</button>
          </footer>
     </section>
</main>

<style>
     :global(body) {
          margin: 0;
          font-family: "Manrope", "Segoe UI", sans-serif;
          background: radial-gradient(
                    circle at 10% 0%,
                    #f6f8ff 0%,
                    #edf1fb 35%,
                    #e6ebf6 100%
               )
               fixed;
          color: #111827;
     }

     .shell {
          min-height: 100vh;
          display: grid;
          place-items: center;
          padding: 1.25rem;
     }

     .panel {
          width: min(880px, 100%);
          background: rgba(255, 255, 255, 0.92);
          border: 1px solid #d6dce8;
          border-radius: 18px;
          box-shadow: 0 14px 42px -28px rgba(15, 23, 42, 0.45);
          padding: clamp(1.25rem, 2vw, 2rem);
          backdrop-filter: blur(4px);
     }

     .header {
          display: flex;
          align-items: flex-start;
          justify-content: space-between;
          gap: 1rem;
          margin-bottom: 1.25rem;
     }

     .eyebrow {
          margin: 0 0 0.35rem;
          font-size: 0.72rem;
          letter-spacing: 0.11em;
          text-transform: uppercase;
          color: #4b5563;
          font-weight: 700;
     }

     h1 {
          margin: 0;
          font-size: clamp(1.45rem, 1.8vw, 1.95rem);
          line-height: 1.15;
          color: #0f172a;
     }

     .primary,
     .ghost {
          border: none;
          border-radius: 10px;
          padding: 0.72rem 1rem;
          font: inherit;
          font-size: 0.92rem;
          font-weight: 700;
          cursor: pointer;
          transition: transform 120ms ease, box-shadow 120ms ease,
               background-color 120ms ease;
     }

     .primary {
          background: #0f172a;
          color: #ffffff;
          min-width: 170px;
     }

     .primary:hover:enabled {
          transform: translateY(-1px);
          box-shadow: 0 8px 18px -12px rgba(15, 23, 42, 0.8);
          background: #1f2937;
     }

     .primary:disabled {
          opacity: 0.65;
          cursor: not-allowed;
     }

     .status-row {
          display: flex;
          align-items: center;
          gap: 0.55rem;
          background: #f8fafc;
          border: 1px solid #e2e8f0;
          border-radius: 10px;
          padding: 0.7rem 0.85rem;
          margin-bottom: 1.1rem;
     }

     .status-row p {
          margin: 0;
          font-size: 0.9rem;
          color: #334155;
          font-weight: 600;
     }

     .dot {
          width: 0.52rem;
          height: 0.52rem;
          border-radius: 50%;
          background: #2563eb;
          box-shadow: 0 0 0 5px rgba(37, 99, 235, 0.12);
          flex: 0 0 auto;
     }

     .grid {
          display: grid;
          gap: 0.85rem;
          grid-template-columns: repeat(2, minmax(0, 1fr));
     }

     .card {
          border: 1px solid #e2e8f0;
          border-radius: 12px;
          background: #ffffff;
          padding: 1rem;
     }

     .card h2 {
          margin: 0;
          font-size: 0.78rem;
          letter-spacing: 0.08em;
          text-transform: uppercase;
          color: #64748b;
          font-weight: 800;
     }

     .card .metric {
          margin: 0.65rem 0 0;
          font-size: clamp(1.05rem, 2.2vw, 1.35rem);
          font-weight: 800;
          color: #0f172a;
     }

     .card small {
          margin-top: 0.45rem;
          display: block;
          color: #64748b;
          font-size: 0.82rem;
     }

     .card-wide {
          grid-column: 1 / -1;
     }

     .mono {
          margin: 0.65rem 0 0;
          color: #0f172a;
          font-family: "JetBrains Mono", "Consolas", monospace;
          font-size: 0.9rem;
          overflow-wrap: anywhere;
          line-height: 1.45;
     }

     .actions {
          margin-top: 1.1rem;
          display: flex;
          justify-content: flex-end;
     }

     .ghost {
          background: #fee2e2;
          color: #9f1239;
     }

     .ghost:hover {
          background: #fecdd3;
     }

     @media (max-width: 700px) {
          .header {
               flex-direction: column;
          }

          .primary {
               width: 100%;
          }

          .grid {
               grid-template-columns: 1fr;
          }

          .card-wide {
               grid-column: auto;
          }
     }
</style>
