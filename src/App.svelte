<script>
  import Upload from "./lib/Upload.svelte";

  import * as dagCbor from "@ipld/dag-cbor";
  import { CID } from "multiformats/cid";
  import * as Block from "multiformats/block";
  import { sha256 } from "multiformats/hashes/sha2";
  import * as ed from "@noble/ed25519";
  import { createHelia } from "helia";
  import { unixfs } from "@helia/unixfs";
  import { fixedSize } from "ipfs-unixfs-importer/chunker";

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
  let mediaUploadStatus;
  let mediaUploadMsg = "";

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

  async function onMediaUpload(evt) {
    mediaUploadStatus = "progress";
    const reader = new FileReader();
    reader.addEventListener(
      "load",
      async () => {
        // @ts-ignore
        let bytes = new Uint8Array(reader.result);
        const helia = await createHelia();
        const fs = unixfs(helia);
        const cid = await fs.addBytes(bytes, {
          // Use kubo default so hashes match
          chunker: fixedSize({ chunkSize: 262144 }),
        });
        if (cid.toString() === vcInfo.fileCid) {
          mediaUploadMsg = "File matches VC ✅";
        } else {
          mediaUploadMsg = "File doesn't match VC ❌";
        }
        mediaUploadStatus = "done";
        helia.gc();
        helia.stop();
      },
      false
    );
    reader.readAsArrayBuffer(evt.detail.file);
  }
</script>

<h1>Authenticated Attributes Verifier</h1>
{#if page === "home"}
  <p class="center">Upload a Verifiable Credentials file to verify</p>
  <div id="upload-vc">
    <Upload on:fileUpload={onVCUpload} />
  </div>
{:else if page === "verify"}
  <hr />
  <article>
    {#if error != null}
      <p class="center">Couldn't verify your file: {error}</p>
    {:else}
      {#if vcInfo.goodSig === true}
        <h2>Valid signature ✅</h2>
      {:else}
        <h2>Invalid signature ❌</h2>
      {/if}
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
              <td>{new Date(vcInfo.ts)}</td>
            </tr>
            <tr>
              <td>Media</td>
              <td><code>{vcInfo.fileCid}</code></td>
            </tr>
          </tbody>
        </table>
      </div>
      <div id="upload-media">
        {#if mediaUploadStatus === "progress"}
          <button
            aria-busy="true"
            aria-label="Please wait…"
            class="secondary"
          />
        {:else if mediaUploadStatus === "done"}
          <p class="center">{mediaUploadMsg}</p>
        {/if}
        <p class="center">Upload a file to see if it matches this claim:</p>
        <div id="upload-media-component">
          <Upload on:fileUpload={onMediaUpload} />
        </div>
      </div>
    {/if}
    <!-- Error or not -->
  </article>
  <button
    on:click={() => {
      page = "home";
      error = null;
      mediaUploadStatus = null;
    }}>Check another</button
  >
{/if}
<footer>
  <hr />
  <p>
    Produced by <a href="https://www.starlinglab.org/">Starling Lab</a>. Source
    code <a href="https://github.com/starlinglab/aa-verify">available</a>.
  </p>
</footer>

<style>
  h1,
  h2,
  .center {
    text-align: center;
  }
  #upload-vc {
    width: 7em;
    height: 7em;
    margin-left: auto;
    margin-right: auto;
  }
  footer {
    margin-top: 2em;
  }
  #upload-media-component {
    width: 3em;
    height: 3em;
    margin-left: auto;
    margin-right: auto;
  }
  #upload-media {
    text-align: center;
  }
</style>
