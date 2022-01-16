<script lang="ts">
  import { candyMachineState, userState } from "../lib/store";
  import { mintOneToken } from "../lib/mint";
  import {
    getUserBalance,
    connectWallet,
    existsOwnerSPLToken,
  } from "../lib/state-helpers";
  import { web3 } from "@project-serum/anchor";
  import { awaitTransactionSignatureConfirmation } from "../lib/connection";
  import confetti from "canvas-confetti";
  import { LAMPORTS_PER_SOL } from "@solana/web3.js";

  const IMAGE_LINK = './src/images/logo.png';

  const txTimeout = 30000;
  const cluster = import.meta.env.VITE_APP_SOLANA_NETWORK?.toString();

  $: date = new Date($candyMachineState?.state.goLiveDate?.toNumber() * 1000);
  $: whitelistPrice =
    $candyMachineState?.state.whitelistMintSettings?.discountPrice;
  $: isActive = $candyMachineState?.state.isActive;
  $: isSoldOut = $candyMachineState?.state.isSoldOut;
  $: userWhitelisted = $userState.isWhiteListed;
  $: price = $candyMachineState?.state.price;

  let isMinting = false;
  let mintSuccessful = false;
  export let solana;
  export let connection;

  async function connectWalletButton() {
    $userState.walletPublicKey = await connectWallet(solana);
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

  async function mint() {
    try {
      isMinting = true;
      if ($candyMachineState?.program && $userState.walletPublicKey) {
        const mint = web3.Keypair.generate();
        const mintTxId = (
          await mintOneToken(
            $candyMachineState,
            new web3.PublicKey($userState.walletPublicKey),
            mint
          )
        )[0];
        let status: any = { err: true };
        if (mintTxId) {
          status = await awaitTransactionSignatureConfirmation(
            mintTxId,
            txTimeout,
            connection,
            "singleGossip",
            true
          );
        }
        if (!status?.err) {
          console.log("Success");
          displaySuccess(mint.publicKey);
        } else {
          console.error("An error occurred");
        }
      }
    } catch (error: any) {
      console.error("An error occurred ", error);
    } finally {
      isMinting = false;
    }
  }

  function displaySuccess(mintPublicKey: web3.PublicKey) {
    // It takes time before state change is cascaded across the network
    // so we update this variables manually upon successful completion
    $candyMachineState.state.itemsRedeemed += 1;
    mintSuccessful = true;
    $userState.solanaExplorerLink =
      cluster == "devnet"
        ? `https://explorer.solana.com/address/${mintPublicKey}?cluster=devnet`
        : `https://explorer.solana.com/address/${mintPublicKey}`;
    throwConfetti();
  }

  function throwConfetti(): void {
    confetti({
      particleCount: 200,
      spread: 70,
      origin: { y: 0.6 },
    });
  }
</script>

<style>
  button{
    font-family: 'Jangkuy', sans-serif;
    font-size: 1.2rem;
    font-weight: 100;
    background-color: transparent;
    border: solid 2px #3DB489;
    justify-content: center;
    align-items: center;
    text-align: center;
  }
  button:hover{
    background-color: #3DB489;
  }

  span{
    font-family: 'Montserrat', sans-serif;
    color: #ffffff;
    font-size: 0.8rem;
    font-weight: 300;
  }
</style>

<div class="flex flex-col">
  {#if !solana}
  <img src={IMAGE_LINK} alt="" class=" w-50 mx-auto m-5" />
    <button
      class=" px-3 py-2 rounded-md hover: text-white font-bold"
      on:click={() => window.open("https://phantom.app/", "_blank")}
      >Get Phantom Wallet</button
    >
  {:else if !$userState.walletPublicKey}
  <img src={IMAGE_LINK} alt="" class=" w-50 mx-auto m-5" />
    <button
      class=" px-3 py-2 rounded-md hover: text-white font-bold"
      on:click={connectWalletButton}>Connect</button
    >
  {:else if !isActive && !userWhitelisted}
  <img src={IMAGE_LINK} alt="" class=" w-50 mx-auto m-5" />
    <button
      class=" px-3 py-2 rounded-md hover: text-white font-bold disabled:bg-gray-400 "
      disabled={true}>Mint live @ {date.toUTCString()}</button
    >
  {:else if isSoldOut}
  <img src={IMAGE_LINK} alt="" class=" w-50 mx-auto m-5" />
    <button
      class=" px-3 py-2 rounded-md hover: text-white font-bold "
      >Sold Out!</button
    >
  {:else if $userState.userBalance < price}
  <img src={IMAGE_LINK} alt="" class=" w-50 mx-auto m-5" />
    <button
      class=" px-3 py-2 rounded-md hover:text-white font-bold disabled:bg-gray-400"
      disabled={true}
      >Insufficient Funds ({(
        (userWhitelisted ? whitelistPrice : price) / LAMPORTS_PER_SOL
      ).toFixed(2)} SOL required)</button
    >
  {:else}
  <img src={IMAGE_LINK} alt="" class=" w-50 mx-auto m-5" />
    <button
      class=" px-3 py-2 rounded-md hover:text-white font-bold disabled:bg-gray-400"
      disabled={isMinting}
      on:click={mint}
    >
      {#if isMinting}
        <span>Minting ...</span>
      {:else if mintSuccessful}
        <span>Mint succesful! Mint another?</span>
      {:else}
        <span
          >Mint ({(
            (userWhitelisted ? whitelistPrice : price) / LAMPORTS_PER_SOL
          ).toFixed(2)} SOL)</span
        >
      {/if}
    </button>
  {/if}
</div>
