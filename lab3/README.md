# Solana Metaplex NFT Lab

This lab will guide you through creating an NFT on Solana using Metaplex, covering the complete process from image upload to NFT minting.

## Prerequisites

- Node.js installed on your system
- A Solana wallet with some SOL on devnet for transaction fees
- Basic understanding of TypeScript/JavaScript

## Lab Overview

Creating an NFT involves three main steps:

1. **Upload the image** to decentralized storage
2. **Upload metadata** that describes the NFT
3. **Create the actual NFT** on the Solana blockchain

## Setup

### 1. Install Dependencies

Install the required packages with pinned minor versions for consistency:

```bash
npm i @solana/web3.js dotenv @metaplex-foundation/mpl-token-metadata @metaplex-foundation/umi @solana-developers/helpers

npm i -D esrun @types/node

npm i @metaplex-foundation/umi-bundle-defaults @metaplex-foundation/umi-uploader-irys
```

### 2. Environment Setup

Create a `.env` file in your project root:

```env
SECRET_KEY=your_base58_private_key_here
```

**Note:** You can get your private key from your Solana wallet. Never share this key or commit it to version control!

## Step 1: Upload the NFT Image

Let's start by creating functionality to upload an image to decentralized storage using Irys.

### What happens under the hood:

- **Irys** is a decentralized storage network that provides permanent data storage
- We convert our local image file into a format suitable for upload
- The upload returns a URI that points to our image on the decentralized network
- This URI will be used later in our NFT metadata

### Create the Image Upload Script

Create `nft-image.ts` with the following structure:

**Step 1a: Setup Wallet and UMI Client**

```typescript
// Load keypair from environment
const kp = getKeypairFromEnvironment("SECRET_KEY");

// Create UMI instance connected to Solana devnet
const umi = createUmi(clusterApiUrl("devnet"));

// Convert keypair and create signer
const keypair = umi.eddsa.createKeypairFromSecretKey(kp.secretKey);
const signer = createSignerFromKeypair(umi, keypair);

// Configure UMI with Irys uploader and signer identity
umi.use(irysUploader());
umi.use(signerIdentity(signer));
```

**Step 1b: Define Image Path and Upload Function**

```typescript
const IMAGE_FILE = "./rug.png";

export async function uploadImage() {
  try {
    console.log("🕣 Uploading image...");

    // Read the image file from disk
    const img = await readFile(IMAGE_FILE);

    // Convert to UMI-compatible format
    const imgConverted = createGenericFile(new Uint8Array(img), "image/png");

    // Upload to Irys and get the URI
    const [myUri] = await umi.uploader.upload([imgConverted]);

    console.log("✅ Done with URI:", myUri);
  } catch (err) {
    console.error("[uploadImage] Failed with error:", err);
  }
}

// Execute the function
uploadImage();
```

### Run the Image Upload

```bash
npx esrun nft-image.ts
```

**Expected output:** You should see a URI like `https://gateway.irys.xyz/[some-hash]`. Save this URI - you'll need it for the next step!

## Step 2: Upload NFT Metadata

Now we'll create and upload the metadata that describes our NFT.

### What happens under the hood:

- **Metadata** is a JSON object that contains information about the NFT (name, description, attributes, etc.)
- It follows the Metaplex metadata standard
- The metadata includes a reference to our uploaded image
- This metadata is also stored on Irys and gets its own URI
- The metadata URI will be used when creating the actual NFT

### Create the Metadata Upload Script

Create `nft-metadata.ts`:

**Step 2a: Setup (Similar to Image Upload)**

```typescript
import "dotenv/config";

import {
  createSignerFromKeypair,
  signerIdentity,
} from "@metaplex-foundation/umi";
import { createUmi } from "@metaplex-foundation/umi-bundle-defaults";
import { getKeypairFromEnvironment } from "@solana-developers/helpers";
import { clusterApiUrl } from "@solana/web3.js";
import { irysUploader } from "@metaplex-foundation/umi-uploader-irys";

const kp = getKeypairFromEnvironment("SECRET_KEY");

const umi = createUmi(clusterApiUrl("devnet"));

const keypair = umi.eddsa.createKeypairFromSecretKey(kp.secretKey);
const signer = createSignerFromKeypair(umi, keypair);

umi.use(irysUploader());
umi.use(signerIdentity(signer));
```

**Step 2b: Define Image URI and Metadata Function**

```typescript
// Replace with YOUR image URI from Step 1
const IMG_URI = "https://gateway.irys.xyz/<HASH>";

async function uploadMetadata() {
  try {
    // Create metadata object following Metaplex standard
    const metadata = {
      name: "Comets RUG",
      symbol: "CRUG",
      desciption: "This is a Stellar RUG",
      image: IMG_URI,
      attributes: [
        { trait_type: "Color", value: "red" },
        { trait_type: "Material", value: "wool" },
        { trait_type: "Size", value: "very big" },
      ],
      properties: {
        files: [{ type: "image/png", uri: IMG_URI }],
      },
    };

    // Upload metadata JSON to Irys
    const metadataUri = await umi.uploader.uploadJson(metadata);

    console.log("✅ Done with metadata URI:", metadataUri);
  } catch (error) {
    console.error("[uploadMetadata] Failed with:", error);
  }
}

uploadMetadata();
```

### Run the Metadata Upload

```bash
npx esrun nft-metadata.ts
```

**Expected output:** Another URI like `https://gateway.irys.xyz/[another-hash]`. Save this metadata URI!

## Step 3: Create the NFT

Finally, we'll create the actual NFT on the Solana blockchain.

### What happens under the hood:

- **NFT Creation** involves creating a new token mint on Solana
- A **mint account** is created to represent the token
- The mint is configured with metadata URI pointing to our uploaded metadata
- **Seller fee basis points** determine royalties in basis points (100 = 1%)
- The process uses Metaplex's Token Metadata Program
- A transaction is sent to the Solana blockchain and confirmed

### Create the NFT Creation Script

Create `nft-create.ts`:

**Step 3a: Extended Imports and Setup**

```typescript
import "dotenv/config";

import {
  createSignerFromKeypair,
  generateSigner,
  percentAmount,
  signerIdentity,
} from "@metaplex-foundation/umi";
import { createUmi } from "@metaplex-foundation/umi-bundle-defaults";
import { getKeypairFromEnvironment } from "@solana-developers/helpers";
import { clusterApiUrl } from "@solana/web3.js";
import { irysUploader } from "@metaplex-foundation/umi-uploader-irys";
import {
  createNft,
  mplTokenMetadata,
} from "@metaplex-foundation/mpl-token-metadata";
import { base58 } from "@metaplex-foundation/umi/serializers";

const kp = getKeypairFromEnvironment("SECRET_KEY");

const umi = createUmi(clusterApiUrl("devnet"));

const keypair = umi.eddsa.createKeypairFromSecretKey(kp.secretKey);
const signer = createSignerFromKeypair(umi, keypair);

// Add Token Metadata Program
umi.use(mplTokenMetadata());
umi.use(signerIdentity(signer));
```

**Step 3b: Define URIs and NFT Creation Function**

```typescript
// Replace with YOUR URIs from previous steps
const IMG_URI = "https://devnet.irys.xyz/<IMAGE_HASH>";
const METADATA_URI = "https://devnet.irys.xyz/<METADATA_HASH>";

async function createMyNft() {
  try {
    // Generate a new keypair for the mint account
    const mint = generateSigner(umi);

    // Create the NFT transaction
    let tx = createNft(umi, {
      name: "Comets RUG",
      mint, // The mint account signer
      authority: signer, // Who has authority over this NFT
      sellerFeeBasisPoints: percentAmount(5),
      isCollection: false, // This is not a collection NFT
      uri: METADATA_URI, // Points to our uploaded metadata
    });

    // Send transaction and wait for confirmation
    let result = await tx.sendAndConfirm(umi);
    const [signature] = base58.deserialize(result.signature);

    console.log("✅ Done! with sig:", signature);
    console.log(`🎉 Your NFT mint address: ${mint.publicKey}`);
  } catch (error) {
    console.error("[createMyNft] Failed with:", error);
  }
}

createMyNft();
```

### Run the NFT Creation

```bash
npx esrun nft-create.ts
```

**Expected output:** A transaction signature and your NFT's mint address!

## Verification

You can verify your NFT was created successfully by:

1. **Checking the transaction** on [Solana Explorer](https://explorer.solana.com/?cluster=devnet)
2. **Viewing your NFT** in a Solana wallet that supports NFTs
3. **Using the mint address** to look up your NFT on NFT marketplaces

## Key Concepts Learned

- **Decentralized Storage**: Using Irys for permanent file storage
- **Metadata Standards**: Following Metaplex metadata format
- **Solana Transactions**: Creating and confirming blockchain transactions
- **Token Mints**: Understanding how NFTs are represented on Solana
- **UMI Framework**: Using Metaplex's unified interface for blockchain interactions

## Troubleshooting

**Common Issues:**

- Make sure you have sufficient SOL in your devnet wallet
- Verify your `SECRET_KEY` is correctly set in `.env`
- Replace placeholder URIs with your actual upload URIs
- Ensure the image file `rug.png` exists in your project directory

## Bonus

- [Update token image](https://developers.metaplex.com/token-metadata/update)
- [Create a collection NFT and verify it](https://developers.metaplex.com/token-metadata/collections)
- [Get all mints in a Collection using DAS API](https://developers.metaplex.com/token-metadata/guides/get-by-collection)
