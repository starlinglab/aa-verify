<script>
  import Upload from "./lib/Upload.svelte";

  let page = "home";
  let vcText;
  let vc;
  let error;
  let vcInfo = {
    pubKey: null,
  };

  function onVCUpload(evt) {
    const reader = new FileReader();
    reader.addEventListener(
      "load",
      () => {
        vcText = reader.result;
        page = "verify";
      },
      false
    );
    reader.readAsText(evt.detail.file);
  }

  function verifyPageSetup() {
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
      }
    } catch (e) {
      error = e;
    }
  }

  $: if (page === "verify") {
    verifyPageSetup();
  }
</script>

<section>
  <h1>Authenticated Attributes Verifier</h1>
  {#if page === "home"}
    <p>Upload a Verifiable Credentials file to verify</p>
    <div id="upload">
      <Upload on:fileUpload={onVCUpload} />
    </div>
    <hr />
    <p>Starling Lab blah blah</p>
  {:else if page === "verify"}
    {#if error != null}
      <p>Couldn't verify your file: {error}</p>
    {:else}
      <p>{vcText}</p>
    {/if}
  {/if}
</section>

<style>
  section {
    text-align: center;
  }
  #upload {
    width: 7em;
    height: 7em;
    margin-left: auto;
    margin-right: auto;
    margin-bottom: 10em;
  }
</style>
