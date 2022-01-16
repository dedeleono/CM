<script lang="ts">
  import { onMount } from "svelte";
  import { Connection } from "@solana/web3.js";
  import { Provider, web3 } from "@project-serum/anchor";
  import { fade } from "svelte/transition";
  import Button from "./components/Button.svelte";
  import Header from "./components/CardHeader.svelte";

  import { candyMachineState, userState } from "./lib/store";
  import {
    getCandyMachineState,
    checkWalletConnected,
    getUserBalance,
    existsOwnerSPLToken,
  } from "./lib/state-helpers";

  /***********************************/
  // Customise the app by changing the following variables.
  const TITLE = "";
  const DESCRTIPTION = "Become a citizen of Shill City";
  const HEADER_TITLE = "sea-shanties.com";
  const HEADER_LINK = "https://sea-shanties.com";
  // Your image or GIF needs to be in the /public folder for this to work
  const IMAGE_LINK = "./src/images/bg.png";
  /***********************************/

  const { solana } = window as any;
  const rpcUrl = import.meta.env.VITE_APP_SOLANA_RPC_HOST?.toString();
  const cluster = import.meta.env.VITE_APP_SOLANA_NETWORK?.toString();
  const candyMachineId = import.meta.env.VITE_APP_CANDY_MACHINE_ID?.toString();
  const opts = { preflightCommitment: "processed" };

  let siteLoading = true;
  let errorOcurred = false;
  let connection: Connection;
  let provider: Provider;
  let candyMachinePublicKey: web3.PublicKey;

  $: itemsRedeemed = $candyMachineState?.state.itemsRedeemed;
  $: itemsAvailable = $candyMachineState?.state.itemsAvailable;

  function checkEnvironmentVariables() {
    // Check if populated
    if (!rpcUrl || !candyMachineId || !cluster) {
      if (!rpcUrl) {
        console.error("RPC URL not populated");
      }
      if (!candyMachineId) {
        console.error("Candy Machine ID not populated");
      }
      if (!cluster) {
        console.error("Environment not populated");
      }
      return true;
    }
    if (candyMachineId.length < 32 || candyMachineId.length > 44) {
      console.error(
        "Candy Machine Public Key is invalid. Enter a length in-between 32 and 44 characters"
      );
      return true;
    }
    return false;
  }

  onMount(async () => {
    // Check if environement variables are populated
    errorOcurred = checkEnvironmentVariables();
    if (errorOcurred) {
      return;
    }

    // If env variables populated, create provider, PK and connection
    connection = new Connection(rpcUrl);
    provider = new Provider(
      connection,
      solana,
      opts.preflightCommitment as web3.ConfirmOptions
    );
    candyMachinePublicKey = new web3.PublicKey(candyMachineId);

    // Get candy machine state
    $candyMachineState = await getCandyMachineState(
      candyMachinePublicKey,
      provider
    );

    // Establish connection to wallet
    if (solana?.isPhantom) {
      $userState.walletPublicKey = await checkWalletConnected(solana);
      if ($userState.walletPublicKey) {
        // Get User Balance
        $userState.userBalance = await getUserBalance(
          $userState.walletPublicKey,
          connection
        );
        // Check if user is whitelisted (ie. check if they have token)
        $userState.isWhiteListed = await existsOwnerSPLToken(
          $userState.walletPublicKey,
          connection,
          $candyMachineState.state.whitelistMintSettings?.mint
        );
      }
    }

    // Stop loading
    siteLoading = false;
  });
</script>
<style>
  main{
    background-image: url("../src/images/out.png");
  }
  a.link{
    font-family: "Scratchy", sans-serif;
    font-size: 1.8rem;
    line-height: 1.2px;
  }
  div.claimed{
    font-family: 'Montserrat', sans-serif;
    color: #ffffff;
    font-size: 0.8rem;
    font-weight: bold;
  }
  div.bg{
    background-color: rgba(0,0,0,0.5);
  }
  div.desc{
    font-family: "Scratchy", sans-serif;
    color: #00698F;
    font-size: 1.5rem;
    font-weight: 300;
    line-height: 1.1px;
    letter-spacing: 1.3px;
  }
</style>
<main class="h-screen">
  <!-- Error section -->
  {#if errorOcurred}
    <div class=" h-full flex">
      <div class="m-auto">
        An error occurred. Please check if your environment variables have been
        populated correctly and redeploy the applcation.
      </div>
    </div>
    <!-- Loading Section -->
  {:else if siteLoading && !errorOcurred}
    <div class=" h-full flex">
      <div class="lds-hourglass m-auto" />
    </div>
  {:else}
    <!-- Menu Bar -->
    {#if HEADER_TITLE}
      <a
        href={HEADER_LINK}
        class="text-white tracking-widest decoration-2 font-mono link"
        >{HEADER_TITLE}</a
      >
    {/if}
    <!-- Card -->
    <div
      class=" max-w-lg mx-auto bg rounded-lg my-12 border-2"
      transition:fade
    >
      <!-- Top Bar -->
      <Header />
      <hr />
      <br />
      <!-- Main Body -->
      <div class="p-6">
        <img src={IMAGE_LINK} alt="" class=" w-50 mx-auto m-5" />
        <div class="text-sm sm:text-md font-semibold pb-5 text-gray-600 desc">
          {DESCRTIPTION}
        </div>
        <Button {solana} {connection} />

        <div class=" tracking-widest text-sm pt-3 text-gray-400 claimed">
          {itemsRedeemed}/{itemsAvailable} claimed
        </div>
        <div class="flex flex-col pt-3">
          {#if $userState.solanaExplorerLink}
            <a
              href={$userState.solanaExplorerLink}
              target="_blank"
              class="text-purple-700 font-semibold link p-1"
              >View on Solana Explorer</a
            >
          {/if}
        </div>
      </div>
    </div>
  {/if}
</main>
