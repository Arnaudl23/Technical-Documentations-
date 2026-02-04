# STP

```arduino

enable
conf t
span vlan id root primary
int range g1/0/14-15 (uniquement sur MLS)
span guard root (uniquement sur MLS)
int range id en fct de la topologie (uniquement sur les Switches)
span portfast (uniquement sur les Switches)
span bpduguard enable (uniquement sur les Switches)
exit

Explications : 
Root guard empêche un port configuré ou désigné de devenir un port racine. Il place le port dans un état ErrDisabled si un bpdu supérieur est reçu. Root guard est activé port par port.
Portfast ne doit être utilisé que sur les ports Edge ou connectés à un host. Si on active portfast sur un port relié à un Switch, on risque de créer une boucle STP.
Un port avec Portfast ne doit jamais recevoir de BPDU sinon il y a un risque de loop. La commande bpduguard permet de mettre le port concerné en état errdisabled à la réception de tout bpdu.
Portfast désactive la génération de notification de topologie (TCN) et fait en sorte que les ports d’accès contournent les états learning et listening et entrent dans l’état transfert.

```

```arduino

enable
conf t
span vlan id root primary
int range g1/0/14-15 (only on MLS)
span guard root (only on MLS)
int range id based on topology (only on switches)
span portfast (only on switches)
span bpduguard enable (only on switches)
exit

Explanations: 
Root guard prevents a configured or designated port from becoming a root port. It places the port in an ErrDisabled state if a superior bpdu is received. Root guard is enabled port by port.
Portfast should only be used on Edge ports or ports connected to a host. If portfast is enabled on a port connected to a Switch, there is a risk of creating an STP loop.
A port with Portfast must never receive a BPDU, otherwise there is a risk of a loop. The bpduguard command allows the port concerned to be set to errdisabled status upon receipt of any bpdu.
Portfast disables the generation of topology change notifications (TCN) and ensures that access ports bypass the learning and listening states and enter the forwarding state.

```