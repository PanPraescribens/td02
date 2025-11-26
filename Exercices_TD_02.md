# Exercices du TD n°1

## Exercice n°1 : reflog HEAD@{n} et
On va juste revenir sur cette notation relative, utile dans certains cas.

### Mise en place
````bash
mkdir td02ex01
cd td02ex01
git init
echo 'Hello world' > main.txt
git add main.txt
git commit -m "Ajoute main"
echo '/secrets/**' > .gitignore
git add .gitignore
git commit -m "AJoute gitnore"
````
Oups ! Le message de commit était un peu de travers. Corrigeons !  
### 1. Correction du commit précédent
````bash
git commit -m "Ajoute .gitignore" --amend
````
### 2. Vérifiez que vous pouvez bien encore voir le commit précédent
````bash
git reflog
# Va vous afficher un truc du genre :
b5b728d (HEAD -> master) HEAD@{0}: commit (amend): Ajoute .gitignore
2f5abb2 HEAD@{1}: commit: AJoute gitnore
````
Remarquez ici la notation absolue (les hash en début de ligne)  
mais aussi la notation relative `HEAD@{1}` qui par exemple vous permet ici de revenir en arrière _dans le temps_.  
````bash
git rev-parse HEAD~1
git rev-parse HEAD@{1}
````
Comparez ces valeurs. `HEAD~1` devrait échouer si vous n'avez pas de commit parent. `HEAD@{1}` vous permet de cibler le commit corrigé, qui en fait existe toujours dans votre dépôt.

### 3. Créons un tag
````bash
git tag ex01
````
Ça nous permet de revenir à notre repo propre si on fait des bêtises par la suite.  

## Exercice n°2 : ORIG_HEAD
Bien pratique quand on fait une manipulation de l'historique et qu'on veut finalement revenir en arrière !  
### 1. Créons un commit erroné
````bash
touch secrets.txt
git add .
git commit -m "Ajoute secrets.txt"
````
Remarquez que secrets.txt a bien été ajouté car vous l'avez créé dans le répertoire racine, pas dans `/secrets`  
Bien sûr c'est une erreur, donc faisons disparaitre ce commit !  
### 2. Revenons en arrière
````bash
git reset --hard HEAD~1
````
Le fichier secrets.txt doit avoir disparu.  
### 3. Revenons vers le futur !
````bash
git reset --hard ORIG_HEAD
````
Et hop! Comme si rien ne s'était passé.
Vérifiez le log et constatez la présence du commit "Ajoute secrets.txt"
Regardez le reflog et constatez la différence.  

> Quels autre manières de revenir en arrière auriez vous pu utiliser ?  

## Exercice n°3 : Première branche, première erreur
### 1. Créons une brnache et faisons des modifications
````bash
git branch -c dev
mkdir window
cd window
echo "Ma première feature !" >> main.txt
git add .
git commit -m "Ajoute une fenêtre"
````
Super ! Sauf que si vous vérifiez maintenant votre historique avec `git log`, vous devriez constater que votre commit est en fait dans la branche master !  
### 2. Corrigeons
````bash
git switch dev
git reset --hard master
#les deux branches pointent maintenant au même endroit
git switch master
git reset --hard ex01
#votre branche master est bien revenue au propre.
````
Vérifiez bien le log de vos branches. Si vous n'êtes pas dans la branche dont vous voulez le log, vous pouvez tapez son nom en argument.  
````bash
git log dev
````
> Quelles autres références pouvions nous utiliser pour faire nos reset ?

## Exercice n°4 : `rebase` - les choses sérieuses commencent !
À ce stade votre répertoire de travail contient :
````bash
# /main.txt
# /secrets/secrets.txt
# si vous êtes sur la branche dev vous avez également 
# /window/main.txt
````

### 1. Rajoutons quelques commits pour notre exercice
````bash
#on travaille sur notre feature
git switch dev
echo 'Les choses avancent bien pour notre nouvelle fenêtre' >> window/main.txt
git add . 
git commit -m "Avance la feature window"

#on doit basculer sur autre chose d'important direcement sur master
git switch master
echo 'On modifie aussi la branche principale pour dautres raisons' > main.txt
git add . 
git commit -m "Hotfix sur master"

#on modifie un peu tant qu'on y est
echo 'Utilise la feature Window1 pour afficher le message message.txt' >> main.txt
echo 'hello world!' > message.txt
git add . 
git commit -m "Refacto sur master"

#on revient sur notre feature
git switch dev
echo 'Ça avançait tellement bien ! Jai perdu le fil maintenant.' >> window/main.txt
git add . 
git commit -m "Évolution sur la feature"
````
### 2. Déplaçons des commits
On décide de déplacer les commits fait sur master dans la feature, en fait.  
En fait dans master on a utilisé la feature qui n'existe pas encore !  
Donc on va tout déplacer.  
````bash
git rebase dev master
````
> Vous allez probablement avoir des conflits car /main.txt a bien changé et git n'arrive pas à deviner quelle version est la bonne.
````bash
CONFLICT (content): Merge conflict in main.txt
error: could not apply 9c5f253... Hotfix sur master
Resolve all conflicts manually, mark them as resolved with
"git add/rm <conflicted_files>", then run "git rebase --continue".
You can instead skip this commit: run "git rebase --skip".
To abort and get back to the state before "git rebase", run "git rebase --abort".
````
Pour résoudre le conflit, suivez les instructions !  
Il va falloir éditer le fichier coupable. Pour les besoins de l'exercice, vous pouvez simplement écraser son contenu en faisant par exemple :  
````bash
echo 'affiche window1 avec message.txt' > main.txt
git add .
git rebase --continue
````
Regardez vos logs pour voir les commits qui y sont incorporés
````bash
git log master
git log dev
````

### 3. Déplaçons nos pointeurs de branche
````bash
git switch dev
git reset --hard master
git switch master
git reset --hard ex02
````
> Normalement ORIG_HEAD nous permet d'annuler la dernière grosse modification, comme un rebase. Mais ici comme nous avons ensuite fait deux reset, ORIG_HEAD ne nous permet pas d'annuler le rebase !  
Sauriez-vous rétablir la situation initiale ?

## Exercice n°5 - Déplaçons une sous-branche !
En pratique, on va avoir une branche master intouchable,  
une branche de dev où tout le monde pousse ses modifs une fois stables  
et chacun se crée une branche de feature pour travailler dans son coin.
Typiquement, on merge un feature dans `dev` et `dev` dans `master`.  
Mais il peut arriver, pour une raison X ou Y qu'on veuille migrer un commit de feature directement sur `master`. `Let's rebase !`  

### Mise en place
Ajoutons quelques commits et une branche.
Revenez à la **racine** du projet puis :
````bash
git switch master
touch some_stuff.txt
git add .
git commit -m "Ajout de stuff"

echo 'This is really useful' > some_stuff.txt
git commit -a -m "Modifie stuff"

git switch dev

git branch -c feat/win
git switch feat/win
echo 'Une feature qui en fait va servir à tout le monde' > toto.txt
git add .
git commit -m "Création d'un utilitaire toto"

git switch dev
echo 'On ajoute des options à notre feature window' >> window/main.txt
git commit -a -m "Développe la feature window"
````

À ce stade on a trois branches, chacune un peu avancée dans son coin
>déplacez la branche feat/win directement sur master