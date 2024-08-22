<script>
  import Upload from "./lib/Upload.svelte";

  import * as dagCbor from "@ipld/dag-cbor";
  import { CID } from "multiformats/cid";
  import * as Block from "multiformats/block";
  import { sha256 } from "multiformats/hashes/sha2";
  import * as ed from "@noble/ed25519";

  let page = "home";
  let vcText;
  let vc;
  let error;
  let vcInfo = {
    goodSig: null,
    val: "",
    attr: "",
    pubKey: "",
    isEncrypted: false,
    fileCid: null,
    ts: "",
  };

  // https://developer.mozilla.org/en-US/docs/Glossary/Base64#the_unicode_problem
  function base64ToBytes(base64) {
    const binString = atob(base64);
    return Uint8Array.from(binString, (m) => m.codePointAt(0));
  }

  async function onVCUpload(evt) {
    const reader = new FileReader();
    reader.addEventListener(
      "load",
      async () => {
        vcText = reader.result;
        await verifyPageSetup();
        page = "verify";
      },
      false
    );
    reader.readAsText(evt.detail.file);
  }

  async function verifyPageSetup() {
    // Validate VC
    try {
      vc = JSON.parse(vcText);
      if (
        JSON.stringify(vc["@context"]) !=
        JSON.stringify([
          "https://www.w3.org/2018/credentials/v1",
          "urn:authattr:vc-schema:1",
        ])
      ) {
        error = "Not a VC, or using unknown schema";
        return;
      }
      if (vc.proof.type != "authattr_ed25519_v1") {
        error = "unknown VC proof type";
        return;
      }
      if (vc.proof.pubKey != vc.issuer.substring(20)) {
        error = "proof public key and issuer don't match";
        return;
      }

      // Assume valid AA VC

      // Extract data
      vcInfo.attr = vc.credentialSubject.attribute;
      vcInfo.ts = vc.issuanceDate;
      vcInfo.fileCid = CID.parse(vc.credentialSubject.id.substring(8));
      vcInfo.pubKey = vc.proof.pubKey;
      switch (vc.credentialSubject.encoding) {
        case "json":
          vcInfo.val = vc.credentialSubject.value;
          break;
        case "base64_dag-cbor":
          vcInfo.val = dagCbor.decode(
            base64ToBytes(vc.credentialSubject.value)
          );
          break;
        case "base64_encrypted_dag-cbor":
          vcInfo.isEncrypted = true;
          break;
        default:
          error = "unknown value encoding type";
          return;
      }

      // Calculate and verify attestation CID
      const attestation = {};
      attestation["CID"] = vcInfo.fileCid;
      attestation["value"] = vcInfo.val;
      attestation["attribute"] = vcInfo.attr;
      attestation["encrypted"] = vcInfo.isEncrypted;
      attestation["timestamp"] = vcInfo.ts;

      console.log(attestation);

      const block = await Block.encode({
        value: attestation,
        codec: dagCbor,
        hasher: sha256,
      });

      if (block.cid.toString() != vc.id.substring(8)) {
        error = "calculated attestation CID doesn't match VC id";
        return;
      }

      // Check signature
      vcInfo.goodSig = await ed.verifyAsync(
        base64ToBytes(vc.proof.sig),
        block.cid.bytes,
        base64ToBytes(vcInfo.pubKey)
      );
    } catch (e) {
      console.log(e);
      error = e;
    }
  }
</script>

<h1>Authenticated Attributes Verifier</h1>
{#if page === "home"}
  <p class="center">Upload a Verifiable Credentials file to verify</p>
  <div id="upload">
    <Upload on:fileUpload={onVCUpload} />
  </div>
{:else if page === "verify"}
  <hr />
  <!-- <article> -->
  {#if error != null}
    <p class="center">Couldn't verify your file: {error}</p>
  {:else}
    {#if vcInfo.goodSig === true}
      <h2>Valid signature ✅</h2>
    {:else}
      <h2>Invalid signature ❌</h2>
    {/if}
    <p>Details:</p>
    <div class="overflow-auto">
      <table class="striped">
        <tbody>
          <tr>
            <td>{vcInfo.attr}</td>
            <td><code>{JSON.stringify(vcInfo.val)}</code></td>
          </tr>
          <tr>
            <td>Signer</td>
            <td><code>{vcInfo.pubKey}</code></td>
          </tr>
          <tr>
            <td>Time</td>
            <td><code>{vcInfo.ts}</code></td>
          </tr>
          <tr>
            <td>Media</td>
            <td><code>{vcInfo.fileCid}</code></td>
          </tr>
        </tbody>
      </table>
    </div>
  {/if}
  <!-- </article> -->
{/if}
<footer>
  <hr />
  <p>Starling Lab blah blah</p>
</footer>

<style>
  h1,
  h2,
  .center {
    text-align: center;
  }
  #upload {
    width: 7em;
    height: 7em;
    margin-left: auto;
    margin-right: auto;
  }
  footer {
    margin-top: 10em;
  }
  article > h1,
  article > h2,
  article > h3,
  article > h4,
  article > h5,
  article > h6 {
    font-size: calc(var(--pico-font-size) * 0.8);
  }
</style>
