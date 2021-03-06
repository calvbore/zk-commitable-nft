# 🏗 Scaffold-ETH - ZK Commitable NFT

> everything you need to build on Ethereum! 🚀

This example builds off of ideas and concepts introduced in the [zk-prove-membership](https://github.com/scaffold-eth/scaffold-eth-examples/tree/zk-prove-membership) build. I recommend going through that example if you have not done so already! :)

```bash
git clone -b zk-commitable-nft https://github.com/scaffold-eth/scaffold-eth-examples.git zk-commitable-nft
cd zk-commitable-nft
yarn install
yarn chain
```

in a second terminal window, start your 📱 frontend:
```bash
cd scaffold-eth
yarn start
```

in a third terminal window, 🛰 deploy your contract:
```bash
cd scaffold-eth
yarn deploy
```

# Circuit

```
packages
├── hardhat
│   ├── circuits
|   │   ├── commit721Tokens
|   |   |   ├── circuit.circom
|   |   |   └── input.json
|   └── powersOfTau28_hez_final_15.ptau
├── react-app
├── services
└── subgraph
```

We have a fairly simple zk circuit to prove our committed token(s). Our circuit makes use of two sparse merkle trees.

The first tree is generated by the smart contract, taking your erc721 token IDs and compressing them into the first merkle root we need.

The second tree is generated by the circuit itself. It takes the token ID(s) we would like to commit to (and keep hidden), places them in the tree and outputs the tree's merkle root.

Let's take a look at what's happening in the circuit.

## parameters

```
template Commit721Tokens(nLevels, nTokens) {

  signal input heldRoot; // defined and provided as arg in smart contract
  signal private input indices[nTokens];
  signal private input ids[nTokens];
  signal private input heldSiblings[nTokens][nLevels + 1];
  signal private input commitNewKeys[nTokens];
  signal private input commitOldKeys[nTokens];
  signal private input commitSiblings[nTokens][nLevels];

  signal output commitRoot;
```

Here we have our template parameters, as well as input and output signals.

The parameter `nLevels` determines the size of our sparse merkle trees, we will be able to support balances up to `nLevels^2`. While `nTokens` will be the number of tokens that we would like to commit to. By default this example has `nLevels` equal to `4`, and `nTokens` equal to `1`.

We have only a single public input signal, `heldRoot`. This will be the merkle root generated using your owned token IDs, and will be calculated and verified by the smart contract.

Next we have a plethora of private inputs.

First we have the `indeces` and `ids` arrays. The `ids` array is the list of erc721 token IDs we will be committing to with the circuit, while the `indeces` array is the key at which those IDs are located in the sparse merkle tree.

Then we have `heldSiblings`, which is all of the sibling nodes needed to verify our IDs are contained in the `heldRoot`.

`commitNewKeys`: The index at which out token IDs will be placed in our newly generated commit merkle root.

`commitOldKeys`: Counterpart to the above signal, it is the sibling leaf node in our use of the sparse merkle tree circuit. See [circomlib](https://github.com/iden3/circomlib) for more details.

`commitSiblings`: The sibling nodes needed to prove a valid insertion to our commit sparse merkle tree.

And finally we have our single output signal, `commitRoot`. This is the root of our second merkle tree containing our commited erc721 token IDs.

## The Verification Tree

If you've gone through the zk-prove-membership branch this snippet of code below should look pretty familiar to you. It is basically the `proveInTree` circuit wrapped in a for loop.

```
// verify token ids are held in the heldRoot
component vTree[nTokens];
for (var i=0; i < nTokens; i++) {
  vTree[i] = SMTVerifier(nLevels+1);
  vTree[i].enabled <== 1;
  vTree[i].root <== heldRoot;
  for (var j=0; j<nLevels+1; j++) vTree[i].siblings[j] <== heldSiblings[i][j];
  vTree[i].oldKey <== 0;
  vTree[i].oldValue <== 0;
  vTree[i].isOld0 <== 0;
  vTree[i].key <== indices[i];
  vTree[i].value <== ids[i];
  vTree[i].fnc <== 0;
}
```

We are taking our `ids` and `indeces` arrays, and using a loop to prove that each ID, at each index, is contained contained within the `heldRoot` sparse merkle tree.

This loop is fairly straightforward, all we are doing is taking the input signals provided for verification and assigning them to an instance of the `SMTVerifier` template found in [circomlib](https://github.com/iden3/circomlib). The verifier checks the inputs are valid and then moves to the next iteration of the loop until all the token IDs are finished up.

## The Commit Tree

Again, the below code snippet should look vaguely familiar to you if you've seen the zk-prove-membership branch. It's the `add2Tree` circuit wrapped in a for loop, yet again (This whole branch is pretty much just smooshing those two circuits into a single circuit).

```
signal newRoots[nTokens + 1];
newRoots[0] <== 0;

component cTree[nTokens];
component rIs0[nTokens];
for (var i=0; i < nTokens; i++) {
  rIs0[i] = IsZero();
  rIs0[i].in <== newRoots[i];

  cTree[i] = SMTProcessor(nLevels);
  cTree[i].oldRoot <== newRoots[i];
  for (var j=0; j<nLevels; j++) cTree[i].siblings[j] <== commitSiblings[i][j];
  cTree[i].oldKey <== commitOldKeys[i];
  cTree[i].oldValue <== 0; // will probably have to be input signal for more than one nTokens
  cTree[i].isOld0 <== rIs0[i].out;
  cTree[i].newKey <== commitNewKeys[i];
  cTree[i].newValue <== ids[i];
  cTree[i].fnc[0] <== 1;
  cTree[i].fnc[1] <== 0;

  newRoots[i+1] <== cTree[i].newRoot;
}

commitRoot <== newRoots[nTokens];
```

First we create a new array of signals, `newRoots`, this is where we will store the new merkle roots calculated by the circuit. We assign the first `newRoots` to be `0` because the tree starts off empty.

Now let's get into the loop!

We'll need to check if the current root is zero so we instantiate an `IsZero` component from [circomlib](https://github.com/iden3/circomlib) as `rIs0` and then we feed it the root calculated by the last insertion we made to the tree.

Next we instantiate a new instance of a tree with `SMTProcessor`, again from [circomlib](https://github.com/iden3/circomlib). The processor can only calculate one insertion at a time, which is why we must instantiate a new instance every loop.

We then assign all of our input signals for our token commitments into their proper places. Notice we assign `rIs0[i].out` to our tree's `isOld0` signal. Explore the [circomlib](https://github.com/iden3/circomlib) codebase to see if you can figure out why.

After all of the input signals have been assigned we take the tree's output, `cTree[i].root`, and assign it to the next member of our `newRoots` signal array. And this loop will run for however many tokens we would like to commit!

At the end of it all we take the last tree's output root and assign it to our circuit's `commitRoot` output signal.

# Contract

Now let me tell you, this is a jenky smart contract and you could probably do a better job at implementing this than me.

This contract uses the poseidon hash function to generate generate a merkle root of an address's held token IDs. Check both `packages/hardhat/deploy/00_deploy_your_contract.js` and [circomlibjs](https://github.com/iden3/circomlibjs) to see how the poseidon hash functions are generated.

The meat of this contract is contained in the `constructRoot` function.

```
function constructRoot(uint256[] memory _data) public view returns(uint256 x) {
      uint256 levelLength = _data.length /*& 1 == 1 ? _data.length + 1 : _data.length*/;
      uint256 currentLevel;

      uint256[] memory data = new uint256[](levelLength);
      for (uint256 i; i < _data.length; i++) {
          data[i] = leafHash(i, _data[i]);
      }
      while (levelLength > 1) {
          unchecked {
              currentLevel += 1;
          }
          if ((levelLength & (levelLength - 1)) == 0) {
              for (uint256 i; i < levelLength/2; i++) {
                  data[i] = nodeHash(data[i], data[i + levelLength/2]);
              }
          } else {
              revert();
          }
          levelLength >>= 1;
      }

      return x = data[0];
  }
```

The function takes an array of `uint256` as it's only argument, and returns a single `uint256`.

First we loop through our array and calculate the leaf hashes of our new tree. Then put our new leaf hashes in a `while` loop to pair them with their sibling leaf nodes and calculate their parent nodes. We do this to each subsequent layer of nodes until we calculate the root hash, and return it as the output of the function.

```
function generateOwnedRoot(address _addr, address _ERC721Contract) public view returns(uint256) {
      uint256 bal = IERC721Enumerable(_ERC721Contract).balanceOf(_addr);
      require((bal & (bal - 1)) == 0); // temp until figure out better root calc
      uint256[] memory ids = new uint256[](bal);
      for (uint256 i; i<bal; i++) {
          ids[i] = IERC721Enumerable(_ERC721Contract).tokenOfOwnerByIndex(_addr, i);
      }
      return constructRoot(ids);
  }
```

The implementation of this function will only work with an array length that is a power of two. If the address has a balance that is not equal to a power of two the function will fail.

We use the `constructRoot` function inside the `generateOwnedRoot` function in order to create a merkle root of each token ID owned by an address. The token must support erc721 enumerable in order for this function to work. We iterate through every token owned by the given address to construct an array and then feed that array into the `constructRoot` function.

We'll then wrap this function inside the `commitHiddenTokens` functions:

```
function commitHiddenTokens(
    address _ERC721Contract,
    uint256[2] memory a,
    uint256[2][2] memory b,
    uint256[2] memory c,
    uint256[2] memory input
) public returns(uint256) {
    uint256 heldRoot = generateOwnedRoot(msg.sender, _ERC721Contract);
    require(heldRoot == input[1], "Invalid.heldRoot");
    uint256 commitRoot = input[0];

    require(
        Verifier.verifyCommit721TokensProof(a, b, c, input) == true,
        "Invalid.Proof"
    );

    addrToCommitment[msg.sender] = commitRoot;
    return commitRoot;
}
```

This function also wraps our proof verifier function. We give this function an erc721 token address and the proof we generate for the tokens we would like to commit to. There is a require statement to enforce the root we give to the circuit is equal to the root generated from the token IDs held by `msg.sender`. Then we verify the proof is valid and assign the circuit's output root as `msg.sender`'s commitment.

That's really all there is to the contract. It's not great, but it put's the circuit on display, and gives a welcoming invitation to make improvements!

# Frontend

Our example front end is currently set up to commit to a single committed erc721 token, with some modification it could support more hidden token IDs in the commitments!

There is a lot going on here in the front end, so we'll only be going through the `generateCommitCalldata` function.

```
async function generateCommitCalldata() {
  const commitKey = Object.keys(commitLeafData)[0];
  // console.log(commitKey);
  // console.log(commitLeafData[commitKey]);

  const commitRes = await blankTree.insert(commitKey, commitLeafData[commitKey]);
  // console.log(commitRes.oldKey);

  const commitInputs = {
    heldRoot: BigInt(heldTree.root.toString()),
    indices: [selectedKey],
    ids: [commitLeafData[commitKey]],
    heldSiblings: [new Array(5).fill(BigInt(0))],
    commitNewKeys: [BigInt(commitKey)],
    commitOldKeys: [commitRes.oldKey],
    commitSiblings: [new Array(4).fill(BigInt(0))]
  }

  const heldProof = await heldTree.find(selectedKey);
  // console.log(heldProof.siblings);

  for (let i = 0; i < 5; i++) {
    // console.log(heldProof.siblings[i]);
    if (heldProof.siblings[i]) {
      commitInputs.heldSiblings[i] = heldProof.siblings[i];
    } else {
      commitInputs.heldSiblings[i] = BigInt(0);
    }
  }

  for (let i = 0; i < 4; i++) {
    if (commitRes.siblings[i]) {
      commitInputs.commitSiblings[i] = commitRes.siblings[i];
    }
  }

  blankTree.delete(commitKey);

  // console.log("proof inputs: ",commitInputs);

  const { proof, publicSignals } = await snarkjs.groth16.fullProve(commitInputs, wasm, zkey);
  const vkey = await snarkjs.zKey.exportVerificationKey(zkey);
  const verified = await snarkjs.groth16.verify(vkey, publicSignals, proof);
  console.log("Proof Verification: ", verified);
  // console.log(publicSignals);
  const proofCaldata = parseGroth16ToSolidityCalldata(proof, publicSignals);

  tx( writeContracts.YourContract.commitHiddenTokens(test721Addr, ...proofCaldata) );
}
```

When this function is called in the frontend it will result in calling our `commitHiddenTokens` function in our smart contract with both an erc721 contract address and a newly generated zk proof of our commitment.

We start the function off by retrieving the key that our token's ID is stored at in the merkle tree.

Next we insert our token ID to an empty merkle tree, `blankTree`(representing the merkle tree we will use for the commitment), in order to assign the data needed to prove an insertion to the `commitRes` constant.

We create a base for the witness, `commitInputs`, we will feed into `snarkjs` to create a proof.

Then we get a proof for the user's selected token ID from the merkle tree containing their owned token IDs and store it in the `heldProof` constant.

```
const heldProof = await heldTree.find(selectedKey);
```

We then loop through `heldProof` to assign each sibling into `commitInputs.heldSiblings`, and then append a `0` to the array, I'm not entirely sure why this is necessary, but the verifier circuit fails to function properly if we don't do this.

```
for (let i = 0; i < 5; i++) {
  // console.log(heldProof.siblings[i]);
  if (heldProof.siblings[i]) {
    commitInputs.heldSiblings[i] = heldProof.siblings[i];
  } else {
    commitInputs.heldSiblings[i] = BigInt(0);
  }
}
```

Then we iterate through a second loop to do the same thing with `commitRes` and `commitInputs.commitSiblings`. This time we do not have to append `0` to the array.

```
for (let i = 0; i < 4; i++) {
  if (commitRes.siblings[i]) {
    commitInputs.commitSiblings[i] = commitRes.siblings[i];
  }
}
```

We delete the `blankTree` commitment data so we can reuse it later for another commitment if we would like.

```
blankTree.delete(commitKey);
```

And finally we use `snarkjs` to generate our zk proof by feeding it `commitInputs` and the circuit's `wasm` and `zkey` files, we generate a `vkey` and verify the proof, parse the proof into solidity calldata with the `parseGroth16ToSolidityCalldata` function, package it all up and call the `commitHiddenTokens` function of our smart contract!

```
const { proof, publicSignals } = await snarkjs.groth16.fullProve(commitInputs, wasm, zkey);
const vkey = await snarkjs.zKey.exportVerificationKey(zkey);
const verified = await snarkjs.groth16.verify(vkey, publicSignals, proof);
console.log("Proof Verification: ", verified);
// console.log(publicSignals);
const proofCaldata = parseGroth16ToSolidityCalldata(proof, publicSignals);

tx( writeContracts.YourContract.commitHiddenTokens(test721Addr, ...proofCaldata) );
```

# Limitations

- **GAS!** This smart contract is going to burn a boatload of gas the larger an address's erc721 balance is.
- The contract will only support making a successful commitment if a user's token balance is equal to a power of two.
- Nothing is keeping an address from transferring a committed token after the commitment has been made.
- Frontend supports commits for only a single token ID.

# challenges

- Modify both the circuit and frontend to support commiting multiple tokens!
- Ensure that a user cannot transfer committed tokens. Try creating a vault per address to transfer NFTs into, they would then be able to commit from tokens held within the vault.
- Modify the repo to work with ERC1155 tokens, then share it with me cause I would love to see this!

---

📣 Make sure you update the `InfuraID` before you go to production. Huge thanks to [Infura](https://infura.io/) for our special account that fields 7m req/day!

# 🏃💨 Speedrun Ethereum
Register as a builder [here](https://speedrunethereum.com) and start on some of the challenges and build a portfolio.

# 💬 Support Chat

Join the telegram [support chat 💬](https://t.me/joinchat/KByvmRe5wkR-8F_zz6AjpA) to ask questions and find others building with 🏗 scaffold-eth!

---

🙏 Please check out our [Gitcoin grant](https://gitcoin.co/grants/2851/scaffold-eth) too!

### Automated with Gitpod

[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#github.com/scaffold-eth/scaffold-eth)
