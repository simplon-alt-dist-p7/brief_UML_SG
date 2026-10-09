# 1 - Comprendre l'existant

## Acteurs

Un acteur "Client" sur le diagramme 

## Echanges et cas alternatifs

>**Tronc Commun**
1. **client &rarr; serveur** : envoie la requête d'inscription avec mail et mot de passe
2. **serveur &rarr; router** : délègue la requête au handler dédié (```router.post("/signup", validateSignup, authController.signup)```
)
3. **router &rarr; middleware de validation** : vérifie la validité du payload de la requête avec zod (```validateSignup()```)


>**Cas alternatif : entrées invalides**
4. **middleware &rarr; client** : rejette la requête et envoie une ```400(Bad Request)```

>**Scénario optimiste : entrées valides**
5. **middleware &rarr; controller** : laisse passer la requête pour la suite du traitement (```next()```)
6. **controller &rarr; service** : lance le processus d'inscription (```signup()```)
7. **service &rarr; ORM** : demande à vérifier si un compte existe avec le mail (```const existing = await userRepository.findByEmail(email)```)
8. **ORM &rarr; base de données** : exécute la recherche d'un utilisateur par mail
9. **base de données &rarr; ORM** :  retourne le résultat (une ligne ou aucune) 
10. **ORM &rarr; service** : transmet l'utilisateur trouvé OU l'absence de résultat 

>**Cas alternatif : mail déjà utilisé**
11. **service &rarr; controller** : signale que l'inscription est impossible car le mail est djéà pris (```throw new EmailAlreadyUsedError()```)
12. **controller &rarr; client** : retourne une (```409(conflict)```) au client

>**Scénario optimiste : mail disponible**

13. **service &rarr; service** : hache le mot de passe(```const passwordHash = await argon2.hash(password);```)
14. **service &rarr; ORM** : demande la création du nouvel utilisateur avec le mot de passe haché (```userRepository.create({ email, passwordHash })```)
15. **ORM &rarr; base de données** : enregistre la nouvelle ligne utilisateur (```repo().save(repo().create(data))```)
16. **base de données &rarr; ORM** : confirme la création et renvoie la ligne enregistrée
17. **ORM &rarr; service** : transmet l'utilisateur créé (```return repo().save(repo().create(data))```)
18. **service &rarr; controller** : renvoie les infos utiles au client(```return { id: user.id, email: user.email }```)
19. **controller &rarr; client** : confirme que l'inscription a réussi (```res.status(200).json({ message: "Inscription réussie" })```)
---

# 2 - Questionner le besoin

"quelles questions posent l'ajout de validation d'email ?" 

- a quel moment on déclenche l'envoi ?
- il se passe quoi une fois le mail validé ? (happy path)
- la non validation du mail empêche le login ? 
- tentative de réinscription avec le même mail avec un compte en attente de confirmation en base ? 
- Le lien de validation expire au bout de combien de temps ? 
- il se passe quoi si le lien est expiré ? 
- possible de demander le renvoi d'un lien de validation ? 
- il se passe quoi si Brevo est down ?
- mail erroné (mail valide dans la forme zod mais faute de frappe ou mail inexistant)
- pas vraiment une question mais ajouter un champ is_confirmed set à false

---

# 3 - Concevoir le diagramme N°1 : Inscription avec envoi de l'email de confirmation (cas nominal)
même diagramme de base et on modifie après l'étape 17 : 

18. **service &rarr; micro-service mail** : demande l'envoi du mail de confirmation (asynchrone email, lien avec token)
19. **service &rarr; controller** : renvoie les infos utiles au client (id, email)
20. **controller &rarr; client** : confirme que l'inscription a réussi (200 created)
 en parrallèle
21. **micro-service mail &rarr; brevo** : appelle Brevo pour déclencher l'envoi
22. **brevo &rarr; micro-service mail** : confirme la réception de la demande

# 4 - Concevoir le diagramme N°2 : Confirmation de l'email au clic sur le lien. (cas nominal)

1. **client &rarr; serveur** : clique le lien reçu par email, on fait un GET /auth/confirm?token=xxx
2. **serveur &rarr; router** : délègue la requête
3. **router &rarr; controller** : transmet la requête au handler de confirmation
4. **controller &rarr; service** : lance la confirmation du compte avec le token reçu
5. **service &rarr; ORM** : demande à retrouver le compte associé à ce token
6. **ORM &rarr; base de données** : exécute la recherche par token
7. **base de données &rarr; ORM** : retourne l'utilisateur correspondant
8. **ORM &rarr; service** : transmet l'utilisateur trouvé
9. **service &rarr; ORM** : demande la mise à jour du compte (passage en statut confirmé)
10. **ORM &rarr; base de données** : exécute la mise à jour
11. **base de données &rarr; ORM** : confirme la mise à jour
12. **ORM &rarr; service** : transmet la confirmation de la mise à jour
13. **service &rarr; controller** : renvoie le succès de l'opération
14. **controller &rarr; client** : confirme que le compte est activé


