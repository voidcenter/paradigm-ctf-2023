# Dragon Tyrant solution (Salute Team)

## Context

The challenge is set in the context of a mini-game. There is [a dragon with excessive strength/constitution (high attack/defense) and 60 HP](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L40-L49). You, [a protagonist with randomly generated and weaker stats and 1 HP](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L122-L130), need to slay the dragon. You and the dragon are both ERC721 tokens and the challenge is solved when the dragon [fails the fight and is subsequently burned](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L185-L189) ([solution check](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/Challenge.sol#L21)). 

You have to initiate the fight and the fight has [at most 256 mini-rounds](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L170). In each round, you and the dragon can [decide either to fight or defend](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L171-L172). [The damage calculation] (https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L192-L198) is summarized here:

<img width="858" alt="Screenshot 2023-10-30 at 4 36 10 PM" src="https://github.com/voidcenter/paradigm-ctf-2023/assets/33541165/eb7b2515-8621-4ad3-a858-71d0778a1d57">

After each mini-round, the damage is subtracted from both sides' HP. Once a party reaches 0 HP, it is burned and the game ends. If both sides reach 0 HP, [the side that initiated the attack --- the player --- is burned](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L185). 

The attack/defense stats of both the dragon and the player are [calculated based on their respective stats and equipment](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L202-L203). Each side can have one weapon and one shield. There are [some pieces of equipment sold in the shop, including a very powerful sword](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/ItemShop.sol#L27-L37). The dragon won't equip anything, while the player starts with [1000 ETH](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/challenge.py#L19). 

There are two places in the game where a random number generator is used. One is [used to determine the player's stats](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L121). The other is [used to determine the dragon's attack/defend decision](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L167). [An ECC-based random number generator](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/Randomness.sol) is used where the seed is [provided from off-chain](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/watcher.py#L68-L69).



## Analysis



