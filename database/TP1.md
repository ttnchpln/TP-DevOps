On crée d'abord notre image, on la build sur le port 

J'ai bien mes données sur la base de données, en allant sur adminer

1-1) Sur le Dockerfile
1-2) Les deux étapes de build permettent d'optimiser le Dockerfile et l'image. Ca permet aussi de réduire la taille de l'image.
L'étape 1, le build stage, permet de compiler l'application Java et créer l'environement MYAPP_HOME
L'étape 2, le run, va lancer l'applcaition Java dans une petite image java 17, dans le meme environement que celui du build, le MYAPP_HOME. ENTRYPOINT permet de dire quel commande lancer au démarrage du container.

