
-------------------------
-------------------------
# Practices

## Pods' philosophy

Il faut les envisager comme des resources _scalables_ et _monitorables_. Il faut aussi considérer la sécurité et le networking. C'est via ces critères que l'on choisit les conteneurs que l'on veut regrouper dans un pod ou pas. 

Ex 1 : si 2 processes doivent scale up en meme temps genre 1 conteneurs applicatif avec un conteneur de monitoring alors on va mettre ces 2 conteneurs dans un pod. 

Ex 2 : si 2 conteneurs partagent les meme règles de sécurité ou bien qu'ils doivent accéder à un service avec la même identité, alors il faut envisager des les déployer dans le meme pod.
