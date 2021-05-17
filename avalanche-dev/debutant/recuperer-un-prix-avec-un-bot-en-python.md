---
description: Comment récupérer le prix d'un actif avec un bot sur un DEX (Pangolin)
---

# Récupérer un prix avec un bot en Python

### Installer web3.py

Exécutez simplement ce code dans votre terminal \(vous devrez peut-être exécuter pip3 en fonction de votre installation\)

```bash
$ pip install web3
```



### Connexion Web3

Nous créons un nouveau fichier: bot.py sur lequel nous importons notre module

{% code title="bot.py" %}
```python
    from web3 import Web3
```
{% endcode %}

Et nous nous connectons à notre fournisseur Web3: 

Ici, j'ai mon propre nœud en cours d'exécution à cette adresse mais vous pouvez en connecter un autre avec [figment.io](https://datahub.figment.io/services/avalanche)

{% tabs %}
{% tab title="Own node" %}
```python
w3 = Web3(Web3.HTTPProvider('http://127.0.0.1:9650/ext/bc/C/rpc'))
```
{% endtab %}

{% tab title="Figment.io" %}
```python
w3 = Web3(Web3.HTTPProvider('https://avalanche--mainnet--rpc.datahub.figment.io/apikey/YOUR_KEY/ext/bc/C/rpc'))
```
{% endtab %}
{% endtabs %}

### 

### Récupération des informations du contrat de la pool

Nous devons d'abord instancier un objet de contrat représentant la pool de liquidité à partir duquel nous interrogeons les données de prix:

```python
liquidityContract = w3.eth.contract(address=address, abi=abi)
```

Dans cet exemple, nous regarderons le prix AVAX et nous prendrons le pool WAVAX / USDT sur pangolin à cette adresse: **0x9ee0a4e21bd333a6bb2ab298194320b8daa26516**

Pour faire des requêtes sur un contrat, nous avons besoin de son ABI, c'est comme une définition de ses fonctions et variables au format JSON. 

Pour le récupérer, vous pouvez aller sur: [https://cchain.explorer.avax.network/address/0x9EE0a4E21bd333a6bb2ab298194320b8DaA26516/contracts](https://cchain.explorer.avax.network/address/0x9EE0a4E21bd333a6bb2ab298194320b8DaA26516/contracts) et cliquer sur "copy ABI".

![](../../.gitbook/assets/image%20%288%29.png)

Vous pouvez le coller sur un nouveau fichier: pool.json

 Maintenant, nous pouvons le lire à partir de notre script python \(nous devons également importer le module json\)

```python
import json

with open("Pool.json") as poolFile :
    poolABI = json.load(poolFile )
```

Nous devons transformer notre adresse en une adresse valide avec Checksum, sinon web3.py ne l'acceptera pas. 

Notre programme ressemble à ceci maintenant:

{% code title="bot.py" %}
```python
from web3 import Web3
import json

w3 = Web3(Web3.HTTPProvider('http://127.0.0.1:9650/ext/bc/C/rpc'))

with open("Pool.json") as poolFile :
    poolABI = json.load(poolFile )

liquidityContract = w3.eth.contract(address=w3.toChecksumAddress("0x9ee0a4e21bd333a6bb2ab298194320b8daa26516"), abi=poolABI)
```
{% endcode %}

Nous devons maintenant passer un appel pour récupérer les réserves de chaque jeton de cette pool

 \(Il existe deux types d'appel à un contrat un qui vient de lire sur la blockchain les informations: on l'appelle avec .call\(\) et un autre qui écrit et interagit réellement avec la blockchain: on l'appelle avec .transact\(\) \)

```python
reserves = liquidityContract.functions.getReserves().call()
reserveToken0 = reserves[0]
reserveToken1 = reserves[1]
```

La fonction renvoie une liste de 3: reserveToken0, reserveToken1 et l'horodatage de la dernière mise à jour.

Notre problème est que nous ne savons pas lequel de token0 ou token1 est WAVAX ou USDT, nous pourrions vérifier manuellement mais nous voulons que notre bot fonctionne même avec d'autres pools dans des commandes inconnues, nous devons donc vérifier lequel est lequel.

```python
token0Address = liquidityContract.functions.token0().call()
token1Address = liquidityContract.functions.token1().call()
```

\*\*\*\*

### Récupération des informations des contrats de jetons

Maintenant que nous avons l'adresse des tokens, nous devons déterminer leur symbol. Pour ce faire, nous devons instancier un contrat pour chacun d'eux et pour cela nous avons besoin de leurs ABI. Nous avons juste besoin d'un ABI ERC20 de base donc nous allons en récupérer un de l'explorateur \(comme ici: [https://cchain.explorer.avax.network/address/0xB31f66AA3C1e785363F0875A1B74E27b85FD66c7/contracts](https://cchain.explorer.avax.network/address/0xB31f66AA3C1e785363F0875A1B74E27b85FD66c7/contracts)\)

Nous copions-collons l'ABI dans ERC20.json et nous le chargeons sur python.

```python
with open("ERC20.json") as erc20File:
    ERC20ABI = json.load(erc20File)
```

Maintenant, nous pouvons créer nos instances de contrats :

```python
token0 = w3.eth.contract(address=w3.toChecksumAddress(token0Address), abi=ERC20ABI)
token1 = w3.eth.contract(address=w3.toChecksumAddress(token1Address), abi=ERC20ABI)
```

And search for symbols and decimals :

```python
token0Symbol = token0.functions.symbol().call()
token0Decimals = token0.functions.decimals().call()

token1Symbol = token1.functions.symbol().call()
token1Decimals = token1.functions.decimals().call()
```



### Code final

{% code title="bot.py" %}
```python
from web3 import Web3
import json

w3 = Web3(Web3.HTTPProvider('http://127.0.0.1:9650/ext/bc/C/rpc'))

with open("pool.json") as poolFile :
    poolABI = json.load(poolFile )
with open("ERC20.json") as erc20File:
    ERC20ABI = json.load(erc20File)

liquidityContract = w3.eth.contract(address=w3.toChecksumAddress("0x9ee0a4e21bd333a6bb2ab298194320b8daa26516"), abi=poolABI)

reserves = liquidityContract.functions.getReserves().call()
reserveToken0 = reserves[0]
reserveToken1 = reserves[1]

token0Address = liquidityContract.functions.token0().call()
token1Address = liquidityContract.functions.token1().call()

token0 = w3.eth.contract(address=w3.toChecksumAddress(token0Address), abi=ERC20ABI)
token1 = w3.eth.contract(address=w3.toChecksumAddress(token1Address), abi=ERC20ABI)

token0Symbol = token0.functions.symbol().call()
token0Decimals = token0.functions.decimals().call()

token1Symbol = token1.functions.symbol().call()
token1Decimals = token1.functions.decimals().call()


if token0Symbol == "WAVAX" :
    price = (reserveToken1/10**token1Decimals) / (reserveToken0/10**token0Decimals)
else :
    price = (reserveToken0/10**token0Decimals) / (reserveToken1/10**token1Decimals)
print("The current price of AVAX is {:4.2f} USDT".format(price))

```
{% endcode %}

\*\*\*\*

**tutoriel créé par** [**Louis**](https://twitter.com/_Syavel_)\*\*\*\*

