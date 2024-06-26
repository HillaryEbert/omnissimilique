<template>
  <v-container>
    <DepositDialog
      v-if="openDepositDialog"
      :selectedName="selectedName"
      :depositWallet="depositWallet"
      :qrCodeText="qrCodeText"
      :depositFee="depositFee"
      :openDepositDialog="openDepositDialog"
      @close="openDepositDialog = false"
    >
    </DepositDialog>
    <v-container v-if="openWithdrawDialog">
      <v-dialog transition="dialog-bottom-transition" max-width="900" v-model="openWithdrawDialog">
        <v-card>
          <v-toolbar color="primary" dark>Withdraw TFT</v-toolbar>
          <v-card-title>
            Interact with the bridge in order to withdraw your TFT to
            {{ selectedName.toUpperCase() }} (withdraw fee is: {{ withdrawFee }} TFT)
          </v-card-title>
          <v-card-text>
            <v-form v-model="isValidSwap">
              <v-text-field
                v-model="target"
                :label="selectedName.toUpperCase() + ' Target Wallet Address'"
                :error-messages="targetError"
                :disabled="validatingAddress"
                :loading="validatingAddress"
                :rules="[() => !!target || 'This field is required', swapAddressCheck]"
              >
              </v-text-field>
              <v-text-field
                @paste.prevent
                label="Amount (TFT)"
                v-model="amount"
                type="number"
                onkeydown="javascript: return event.keyCode == 69 || /^\+$/.test(event.key) ? false : true"
                :rules="[
                  () => !!amount || 'This field is required',
                  () =>
                    (amount.toString().split('.').length > 1 ? amount.toString().split('.')[1].length <= 3 : true) ||
                    'Amount must have 3 decimals only',
                  () => amount >= 2 || 'Amount should be at least 2 TFT',
                  () => amount < parseFloat($store.state.credentials.balance.free) || 'Amount cannot exceed balance',
                ]"
              ></v-text-field>
            </v-form>
          </v-card-text>
          <v-card-actions class="justify-end">
            <v-btn @click="openWithdrawDialog = false" color="grey lighten-2 black--text">Close</v-btn>
            <v-btn
              class="primary white--text"
              @click="withdrawTFT(target, amount)"
              :disabled="!isValidSwap || validatingAddress"
              :loading="loadingWithdraw"
              >Submit</v-btn
            >
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-container>
    <v-card color="primary" class="pa-5 my-5 white--text">
      <h3 class="text-center">Transfer TFT Across Chains</h3>
    </v-card>
    <v-card class="pa-5 my-5">
      <v-select
        :items="items"
        label="Please select a chain:"
        v-model="selectedItem.item_id"
        item-text="name"
        item-value="id"
      >
      </v-select>
    </v-card>
    <v-container class="d-flex justify-center pa-5 my-2">
      <v-btn color="primary" class="white--text mx-5 pa-5" @click="openDepositDialog = true">deposit</v-btn>

      <v-btn color="primary" class="mx-5 pa-5 white--text" @click="openWithdrawDialog = true">withdraw</v-btn>
    </v-container>

    <v-container class="d-flex justify-center pa-5 my-3">
      <a
        color="primary"
        :target="'blank'"
        class="text-decoration-none"
        :href="'https://manual.grid.tf/threefold_token/tft_bridges/tfchain_stellar_bridge.html#how-to-use-the-tfchain-stellar-bridge'"
        >How is it done?</a
      >
    </v-container>
  </v-container>
</template>

<script lang="ts">
import { default as StellarSdk, StrKey } from "stellar-sdk";
import { Component, Vue } from "vue-property-decorator";

import DepositDialog from "../components/DepositDialog.vue";
import config from "../config";
import { balanceInterface, getBalance } from "../lib/balance";
import { getDepositFee, getWithdrawFee, withdraw } from "../lib/swap";

@Component({
  name: "TransferView",
  components: { DepositDialog },
})
export default class TransferView extends Vue {
  openDepositDialog = false;
  openWithdrawDialog = false;
  selectedItem = {
    item_id: 1,
  };
  selectedName = "";
  isValidSwap = false;
  items = [{ id: 1, name: "stellar" }];
  $api: any;
  depositWallet = "";
  depositFee = 0;
  qrCodeText = "";
  withdrawFee = 0;
  amount = 0;
  target = "";
  loadingWithdraw = false;
  targetError = "";
  server = new StellarSdk.Server(config.horizonUrl);
  validatingAddress = false;

  mounted() {
    if (this.$api && this.$store.state.credentials.initialized) {
      if (config.bridgeTftAddress) {
        this.depositWallet = config.bridgeTftAddress;
      }
      this.qrCodeText = `TFT:${this.depositWallet}?message=twin_${this.$store.state.credentials.twin.id}&sender=me`;
      this.selectedName = this.items.filter(item => item.id === this.selectedItem.item_id)[0].name;
    } else {
      this.$router.push({
        name: "accounts",
        path: "/",
      });
    }
  }

  async updated() {
    this.selectedName = this.items.filter(item => item.id === this.selectedItem.item_id)[0].name;
    this.target;
    this.amount;
    this.depositFee = await getDepositFee(this.$api);
    this.withdrawFee = await getWithdrawFee(this.$api);
  }

  unmounted() {
    this.$store.commit("UNSET_CREDENTIALS");
  }

  swapAddressCheck() {
    this.targetError = "";
    if (!this.target) return true;
    const isValid = StrKey.isValidEd25519PublicKey(this.target);
    const blockedAddresses = [
      "GBNOTAYUMXVO5QDYWYO2SOCOYIJ3XFIP65GKOQN7H65ZZSO6BK4SLWSC",
      "GA2CWNBUHX7NZ3B5GR4I23FMU7VY5RPA77IUJTIXTTTGKYSKDSV6LUA4",
    ];
    if (blockedAddresses.includes(this.target)) {
      this.targetError = "Blocked Address";
      return false;
    }
    if (!isValid || this.target.match(/\W/)) {
      this.targetError = "invalid address";
      return false;
    }
    this.targetError = "";
    this.validatingAddress = true;
    this.isValidSwap = false;
    if (this.selectedName == "stellar") this.validateAddress();
    return true;
  }
  async validateAddress() {
    try {
      // check if the account provided exists on stellar
      const account = await this.server.loadAccount(this.target);
      // check if the account provided has the appropriate trustlines
      const includes = account.balances.find(
        (b: { asset_code: string; asset_issuer: string }) =>
          b.asset_code === "TFT" && b.asset_issuer === config.tftAssetIssuer,
      );
      if (!includes) throw new Error("invalid trustline");
    } catch (e) {
      this.targetError =
        (e as Error).message === "invalid trustline"
          ? "Address does not have a valid trustline to TFT"
          : "Address not found";
      this.validatingAddress = false;
      this.isValidSwap = false;
      return;
    }
    this.validatingAddress = false;
    return;
  }
  public async withdrawTFT(target: string, amount: number) {
    this.loadingWithdraw = true;
    withdraw(
      this.$store.state.credentials.account.address,
      this.$api,
      target,
      amount,
      (res: { events?: never[] | undefined; status: any }) => {
        console.log(res);
        if (res instanceof Error) {
          console.log(res);
          this.loadingWithdraw = false;
          return;
        }
        const { events = [], status } = res;
        console.log(`Current status is ${status.type}`);
        switch (status.type) {
          case "Ready":
            this.$toasted.show(`Transaction submitted`);
        }
        if (status.isFinalized) {
          console.log(`Transaction included at blockHash ${status.asFinalized}`);
          if (!events.length) {
            this.$toasted.show("Withdraw failed!");
            this.openWithdrawDialog = false;
          } else {
            // Loop through Vec<EventRecord> to display all events
            events.forEach(({ phase, event: { data, method, section } }) => {
              console.log(`\t' ${phase}: ${section}.${method}:: ${data}`);
              if (section === "tftBridgeModule" && method === "BurnTransactionCreated") {
                this.$toasted.show("Withdraw submitted!");
                this.openWithdrawDialog = false;
                getBalance(this.$api, this.$store.state.credentials.account.address).then(
                  (balance: balanceInterface) => {
                    this.$store.state.credentials.balance.free = balance.free;
                  },
                );
                this.target = "";
                this.amount = 0;
                this.loadingWithdraw = false;
              } else if (section === "system" && method === "ExtrinsicFailed") {
                this.$toasted.show("Withdraw failed!");
                this.openWithdrawDialog = false;
                this.target = "";
                this.amount = 0;
                this.loadingWithdraw = false;
              }
            });
          }
        }
      },
    ).catch(err => {
      console.log(err.message);
      this.$toasted.show("Withdraw failed!");
      this.openWithdrawDialog = false;
      this.target = "";
      this.amount = 0;
      this.loadingWithdraw = false;
    });
  }
}
</script>

<style scoped>
.theme--dark.v-application a {
  color: white;
}
</style>
