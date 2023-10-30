# Dragon Tyrant writeup (Salute Team)

## Context

The challenge is set in the context of a mini-game. There is [a dragon with excessive strength/constitution (high attack/defense) and 60 HP](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L40-L49). You, [a protagonist with randomly generated and weaker stats and 1 HP](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L122-L130), need to slay the dragon. You and the dragon are both ERC721 tokens and the challenge is solved when the dragon [fails the fight and is subsequently burned](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L185-L189) ([solution check](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/Challenge.sol#L21)). 

You have to initiate the fight and the fight has [at most 256 mini-rounds](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L170). In each round, you and the dragon can [decide either to fight or defend](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L171-L172). [The damage calculation] (https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L192-L198) is summarized here:

<img width="858" alt="Screenshot 2023-10-30 at 4 36 10 PM" src="https://github.com/voidcenter/paradigm-ctf-2023/assets/33541165/eb7b2515-8621-4ad3-a858-71d0778a1d57">

After each mini-round, the damage is subtracted from both sides' HP. Once a party reaches 0 HP, it is burned and the game ends. If both sides reach 0 HP, [the side that initiated the attack --- the player --- is burned](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L185). 

The attack/defense stats of both the dragon and the player are [calculated based on their respective stats and equipment](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L202-L203). Each side can have one weapon and one shield. There are [some pieces of equipment sold in the shop, including a very powerful sword](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/ItemShop.sol#L27-L37). The dragon won't equip anything, while the player starts with [1000 ETH](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/challenge.py#L19). 

There are two places in the game where a random number generator is used. One is [used to determine the player's stats](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L121). The other is [used to determine the dragon's attack/defend decision](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L167). [An ECC-based random number generator](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/Randomness.sol) is used where the seed is [provided from off-chain](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/watcher.py#L68-L69).



## Analysis

### Attack/Defend decision

To kill the dragon, we need to reduce its HP to 0 without losing our only HP. Looking at the attack/defend matrix, this means that we can't let the attack-attack scenario happen. In an attack-attack round, both sides' HP will be reduced to 0 resulting in the player's defeat. This happens because both sides' attack_stats are much higher than the other party's HP. Because a defend-defend round is like a NOP, we can only rely on the attack-defend rounds and the defend-attack rounds.

<img width="854" alt="Screenshot 2023-10-30 at 4 57 58 PM" src="https://github.com/voidcenter/paradigm-ctf-2023/assets/33541165/19f65011-e120-4816-a8e4-39da1bc24ebf">

Because both parties' attack/defend decisions are [committed upfront](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L161-L167), we would need to know the dragon's choices beforehand to avoid the attack-attack rounds. This requires us to predict [random number generation](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/Randomness.sol#L58) which in turn requires us to predict [the random seed](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/watcher.py#L69). Fortunately, there is [a python library](https://github.com/tna0y/Python-random-module-cracker) that can predict the output of python's random module once it has observed about 20k bits of output from the generator.

So how can we feed this library 20k bits of output from the python random module that we have no access to? It turns out that we can [mint arbitrary number of players](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L57) and each mint transaction would [trigger the off-chain seed provider to submit a random seed](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L63). We can monitor such transactions in the pending transaction pool and therefore capture the seeds. Indeed, we found that we can predict the random seeds after capturing the seeds from 78 mints: 

    predicting next random bytes 0xa1d6cc6a29dfdc86fc1804c2ecae46695b73f7cc4830f524179270531e62252c
    new transaction: 0xc13075d71437d66e4c09841ce26fccc04acbbc55a1d76902c2265615c7d076a8
    actual random bytes =  0xa1d6cc6a29dfdc86fc1804c2ecae46695b73f7cc4830f524179270531e62252c 

Because the ECC random number generator is deterministic, we can predict the dragon's attack/defense decision. We always do the opposite of what the dragon does. If the dragon attacks, we defend. If the dragon defends, we attack.


### Attack/defend stats

Without any equipment, our modest stats would lead to our defeat in the battle. The dragon has [`type(uint40).max`](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L42) attack_stats and [`type(uint40).max - 1`](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L44) defense_stats. Without any equipment, we would not cause any damage to the dragon when we attack and the dragon defends. When the dragon attacks and we defend, we would lose immediately. 

Naturally, we set our sights on [the legendary sword](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/ItemShop.sol#L36). With this sword, our attack_stats would reach `type(uint40).max`, allowing us to inflict 1 HP damage on the dragon when we attack and the dragon defends. If we repeat this 60 times, the dragon dies. There is hope.

How can we afford the sword, which costs 1 million ETH, with only 1,000 ETH? It turns out that when we [equip the sword](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L67), the game lets us pass in the shop contract by ourselves and would happily proceed as long as the shop contract [has been previously approved by the factory contract's owner](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/Factory.sol#L51). Upon closer inspection, such checking is not done by verifying the shop contract address but by [comparing the shop contract's codehash](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/Factory.sol#L43-L48). This means that as long as we pass in a shop contract with the same codehash, we would be able to proceed. Because `extcodehash` doesn't include the constructor, we could potentially create our own item shop with the same code but a different constructor, and use that to equip swords to our player.

This works. Using a fake shop with the following constructor, we are able to obtain the legendary swords as well as a new legendary shield: 

    constructor() {
        factory = IFactory(msg.sender);
        _mint(msg.sender, 3, 1, "");
    
        _itemInfo[4] =
            ItemInfo({name: "Legendary Shield", slot: EquipmentSlot.Shield, value: type(uint40).max, price: 1});
        _mint(msg.sender, 4, 1, "");
    }

With both pieces of the legendary equipment in hand, we achieve `type(uint40).max` attack_stats and `type(uint40).max` defense_stats. We will not lose any HP when the dragon attacks and we will cause 1 HP damage to the dragon when we attack. 


## Solution

Here is the step-by-step solution:

1. Mint the player token to our own wallet.
2. Deploy the fake item shop and use it to equip the player with both pieces of the legendary equipment.
3. Deploy [the attacker contract](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L161), as required by the challenge. This contract will take ownership of the player token, [initiate the fight](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L79), and provide the player's attack/defend decisions.
4. Transfer the player token to the attacker contract.
5. Start the pending transaction pool listener, which monitors the [resolveRandomness](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/NFT.sol#L97) transactions. It captures the seeds and predicts the next seed once it has gathered sufficient information.
6. Mint 78 additional player tokens.
7. By now, the pool listener should have gathered enough information to predict the next seed.
8. Feed the predicted seed into the [random number generator](https://github.com/voidcenter/paradigm-ctf-2023/blob/main/dragon-tyrant/challenge/project/src/Randomness.sol#L58) to determine the dragon's attack/defend decisions.   
9. Invert the dragon's decision string bit-wise to derive the player's decisions. When queried, the attacker contract will provide the player's decisions.
10. The attacker contract initiates the attack, leading to the dragon's defeat.

