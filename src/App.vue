<template>
  <v-app>
    <v-main>
      <div id="content">
        <div id="info-text">
          <p>You sent your hard-earned SOL to a token account address? 🤦🥲</p>
          <p>This app helps you get it back! 🥳</p>
        </div>
        <v-form v-model="validForm" id="form" @submit.prevent="onSubmit">
          <v-text-field
            v-model="tokenAccountAddress"
            :rules="pubkeyRules"
            label="The Token Account Address you accidentally sent your SOL to"
            clearable
          ></v-text-field>
          <v-select v-model="wallet" :items="wallets" label="Wallet"></v-select>
          <v-btn class="mr-4" type="submit" :disabled="!validForm">
            Connect & Submit
          </v-btn>
        </v-form>
      </div>
    </v-main>
  </v-app>
</template>

<script>
import {
  Account,
  clusterApiUrl,
  Connection,
  PublicKey,
  SystemProgram,
  Transaction
} from "@solana/web3.js";
import {
  AccountLayout,
  TOKEN_PROGRAM_ID,
  Token,
  ASSOCIATED_TOKEN_PROGRAM_ID
} from "@solana/spl-token";
import Wallet from "@project-serum/sol-wallet-adapter";

const getMintPubkeyFromTokenAccountPubkey = async (
  tokenAccountPubkey,
  connection
) => {
  try {
    const tokenMintData = (
      await connection.getParsedAccountInfo(tokenAccountPubkey, "confirmed")
    ).value.data;
    const tokenMintAddress = tokenMintData.parsed.info.mint;

    return new PublicKey(tokenMintAddress);
  } catch (err) {
    throw new Error(
      "Error calculating mint address from token account. Are you sure you inserted a valid token account address"
    );
  }
};

const walletMap = {
  sollet: "https://www.sollet.io",
  bonfida: "",
  solflare: ""
};

export default {
  name: "App",

  data: () => ({
    wallets: ["sollet" /* , "bonfida", "solflare" */],
    wallet: "sollet",
    validForm: true,
    tokenAccountAddress: "",
    pubkeyRules: [
      v => !!v || "Address is required",
      v => {
        try {
          new PublicKey(v);
          return true;
        } catch {
          return "Invalid Address";
        }
      }
    ]
  }),
  methods: {
    async onSubmit() {
      const connection = new Connection(
        clusterApiUrl("mainnet-beta"),
        "confirmed"
      );
      const wallet = new Wallet(walletMap[this.wallet], "mainnet-beta");
      await wallet.connect();

      const tokenPublicKey = new PublicKey(this.tokenAccountAddress);

      let tokenMintPublicKey;
      try {
        tokenMintPublicKey = await getMintPubkeyFromTokenAccountPubkey(
          tokenPublicKey,
          connection
        );
      } catch (err) {
        console.error(err.message);
        return;
        // TODO: show notification
      }
      const newTokenKeypair = new Account();

      const tokenAccountMinimumRent = await connection.getMinimumBalanceForRentExemption(
        AccountLayout.span,
        "confirmed"
      );

      const ixs = [];

      const createTokenAccountIx = SystemProgram.createAccount({
        fromPubkey: wallet.publicKey,
        newAccountPubkey: newTokenKeypair.publicKey,
        lamports: tokenAccountMinimumRent,
        space: AccountLayout.span,
        programId: new PublicKey(TOKEN_PROGRAM_ID.toBase58()) // just TOKEN_PROGRAM_ID fails :/
      });
      ixs.push(createTokenAccountIx);

      const initTokenAccountIx = Token.createInitAccountInstruction(
        TOKEN_PROGRAM_ID,
        tokenMintPublicKey,
        newTokenKeypair.publicKey,
        wallet.publicKey
      );
      ixs.push(initTokenAccountIx);

      const tokensInTokenAccount = (
        await connection.getParsedAccountInfo(tokenPublicKey)
      ).value.data.parsed.info.tokenAmount.amount;

      const transferTokensIx = Token.createTransferInstruction(
        TOKEN_PROGRAM_ID,
        tokenPublicKey,
        newTokenKeypair.publicKey,
        wallet.publicKey,
        [],
        tokensInTokenAccount
      );
      ixs.push(transferTokensIx);

      const closeTokenAccountIx = Token.createCloseAccountInstruction(
        TOKEN_PROGRAM_ID,
        tokenPublicKey,
        wallet.publicKey,
        wallet.publicKey,
        []
      );
      ixs.push(closeTokenAccountIx);

      const associatedTokenAddress = await Token.getAssociatedTokenAddress(
        ASSOCIATED_TOKEN_PROGRAM_ID,
        TOKEN_PROGRAM_ID,
        tokenMintPublicKey,
        wallet.publicKey
      );

      const associatedTokenData = await connection.getParsedAccountInfo(
        associatedTokenAddress
      );

      if (
        !associatedTokenData.value ||
        associatedTokenAddress.equals(tokenPublicKey)
      ) {
        const createAssoIx = Token.createAssociatedTokenAccountInstruction(
          ASSOCIATED_TOKEN_PROGRAM_ID,
          TOKEN_PROGRAM_ID,
          tokenMintPublicKey,
          associatedTokenAddress,
          wallet.publicKey,
          wallet.publicKey
        );
        ixs.push(createAssoIx);
      }

      const transferTokensBackIx = Token.createTransferInstruction(
        TOKEN_PROGRAM_ID,
        newTokenKeypair.publicKey,
        associatedTokenAddress,
        wallet.publicKey,
        [],
        tokensInTokenAccount
      );
      ixs.push(transferTokensBackIx);

      const closeNewTokenAccountIx = Token.createCloseAccountInstruction(
        TOKEN_PROGRAM_ID,
        newTokenKeypair.publicKey,
        wallet.publicKey,
        wallet.publicKey,
        []
      );
      ixs.push(closeNewTokenAccountIx);

      const tx = new Transaction();
      tx.add(...ixs);
      tx.feePayer = wallet.publicKey;
      tx.recentBlockhash = (
        await connection.getRecentBlockhash("finalized")
      ).blockhash;
      const signed = await wallet.signTransaction(tx);
      signed.partialSign(newTokenKeypair);
      const txid = await connection.sendRawTransaction(signed.serialize(), {
        skipPreflight: false,
        preflightCommitment: "confirmed"
      });
      await connection.confirmTransaction(txid, "confirmed");
      console.log("success!");
    }
  }
};
</script>

<style lang="scss" scoped>
#content {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

#info-text {
  margin-top: 45px;
}

#form {
  margin-top: 10px;
  width: 600px;
}
</style>
