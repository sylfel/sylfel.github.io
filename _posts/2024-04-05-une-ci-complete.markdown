---
layout: post
title: Une CI complète
date: 2024-04-05T12:44:38.824Z
categories: Test
draft: false
tags:
  - Github action
  - Laravel
---

# Pourquoi une CI "complète" ?

Sur un projet personnel, j'utilise [Livewire](https://livewire.laravel.com/) et lors de la création d'un composant, l'outil permet de créer un simple test qui va juste vérifier que le composant peut se créer et s'afficher.  
Dans mon cas, c'est un code très simple :

```php
<?php

use App\Livewire\Charts;
use Livewire\Livewire;

it('renders successfully', function () {
    Livewire::test(Charts::class)
        ->assertStatus(200);
});
```

Je test en local, tout fonctionne.  
J'ajoute du code dans mon composant, je test, ça fonctionne.  
Je pousse sur github, où une action va me lancer mes tests, et là... erreur !

Après une rapide recherche, je trouve le coupable, la configuration de la CI

## C'est quoi une CI ?

Une CI (*Continuous Integration*, intégration continue en bon français) représente tout le processus d'automatisation des tests, de la mise en forme, des vérifications du code qu'un développeur va produire.

Une bonne CI à différents buts :
- **homogénéiser** le code, via des formatteurs tel que [Pint](https://github.com/laravel/pint) (pour php) ou [Prettier](https://prettier.io/) (pour plein de langage)
- **améliorer** la qualité du code via des linters tel que [PhpStan](https://phpstan.org/) (pour php) ou [EsLint](https://eslint.org/) (Ts/Js)
- **valider** des tests unitaires / tests d'intégrations via des testeurs tel que [Pest](https://pestphp.com/)

## Pourquoi les tests n'ont pas fonctionnés ?

Le composant, dans son initialisation, éxecute une requête SQL.  
Et c'était la première requête exécutée dans mes test *(c'est le début du projet...)*  
Erreur donc, car la requête plantait.

Et elle plantait car il n'y avait pas de service pour exécuter la requête !

*Une autre erreur aussi, c'est que pour un tel test, une séparation composant / requête devrait être faite, je reviendrais dessus un autre jour*

## Solution

J'ai donc configurer un service dans les actions github pour que les tests puissent s'exécuter.  
La configuration de mon action ressemble donc à ceci :

```yaml
{% raw %}name: Test

on:
  pull_request:
    branches: [ "main" ]

jobs:
  laravel-tests:
    runs-on: ubuntu-latest
    env:
      MYSQL_ROOT_PASSWORD: root_testing
      MYSQL_DB: testing
      MYSQL_USER: testing
      MYSQL_PASSWORD: testing
    services:
      mysql:
        image: mysql:latest
        env:
          MYSQL_ROOT_PASSWORD: ${{ env.MYSQL_ROOT_PASSWORD }}
          MYSQL_DATABASE: ${{ env.MYSQL_DB }}
          MYSQL_USER: ${{ env.MYSQL_USER }}
          MYSQL_PASSWORD: ${{ env.MYSQL_PASSWORD }}
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3{% endraw %}
```

Avec ceci, une base de donnée MySql est lancée sur le port **3306**, avec une base nommée **testing**, pour un utilisateur **testing** et un mot de passe **testing** (il y a comme un truc redondant, peut-être même sporadique...)

Dans mon action, je n'ai plus qu'à lancer les tests :
```yaml
{% raw %}  - name: Execute tests (Unit and Feature tests) via Artisan command
      env:
        DB_DATABASE: ${{ env.MYSQL_DB }}
        DB_USERNAME: ${{ env.MYSQL_USER }}
        DB_PASSWORD: ${{ env.MYSQL_PASSWORD }}
      run: php artisan test{% endraw %}
```
Et maintenant, tout fonctionne proprement.  
Est-ce que tout est parfait ?  

Si seulement...

La suite dans un prochain post !