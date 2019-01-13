# ElasticSearch ressources pour l'IUT

Commencez par cloner ou telecharger ce repository. Puis démarrer ElasticSearch

```bash
docker-compose up elasticsearch
```

et testez si tout s'est bien passé :

```bash
curl http://localhost:9200
```

Documentation pour la version utilisée : [Elastic Online Guide](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/index.html)

## Concepts et environnement d'ElasticSearch

> **Exercice**:
>
> 1.  Comment se nomme le cluster ?
>
> 2.  De combien de noeuds est-il composé ? Quels sont leur nom ?
>
> 3.  Quel est le status du cluster ? `curl http://localhost:9200/_cluster/health`
>
> 4.  Créez un index portant "votre nom-movies" avec la commande `curl -XPUT http://localhost:9200/name-movies`
>
> 5.  Combien de shards utilise votre index ?
>
> 6.  Et maintenant, quel est le status du cluster ? L'expliquer
>
> 7.  Recréez l'index pour que le status du cluster soit stable (NB: utiliser l'argument `-H 'Content-Type: application/json'` dans la commande curl)

## Kibana

Kibana est l'outil d'elastic pour interroger, visualiser et créer des dashboards autour des données stockées dans ElasticSearch.

Pour éviter la répétition des commandes curl pour la suite, nous allons utiliser le `dev tools` de Kibana.

Pour cela, démarrer le container avec la commande `docker-compose up kibana` et aller ensuite sur la page [http://localhost:5601/app/kibana#/dev_tools/](http://localhost:5601/app/kibana#/dev_tools/).

## CRUD

### Inserer un film

```bash
PUT name-movies/movie/99000
{
    "genres": ["Sci-Fi","IMAX"],
    "title": "Interstellar",
    "year": 2014
}
```

### Récupérer un film

```bash
GET name-movies/movie/99000
```

### Modifier un film

```bash
POST name-movies/movie/99000/_update
{
    "doc": {
        "title": "Interstellar -- THE MOVIE!"
    }
}
```

Vous pouvez voir dans le retour de la commande que le resultat est `updated`, mais aussi que la version a été incrémentée.

### Supprimer un film

```bash
DELETE name-movies/movie/99000
```

Vous voyez dans le résultat que la version a encore été incrémentée.

> Que peut-on en déduire ?
>
> Que va-t-il se passer lorsque je vais de nouveau créer un film avec cet id ?

## Mapping & analyzer

### Mapping

Vous pouvez vérifier le mapping actuel de votre index avec la commande `curl http://localhost:9200/name-movies/_mapping`.

Nous allons maintenant nous servir d'une base de films. Pour cela, ouvrez le fichier `movies.json` et remplacez `"_index": "movies"` avec le nom de votre index partout dans le fichier.

Puis nous allons importer les données avec la ligne de commande suivante:

`curl -H "Content-Type: application/json" -XPUT 'http://localhost:9200/_bulk?pretty' --data-binary @movies.json`

> A quoi ressemble le mapping ? Vous semble-t-il correct ? Que pourrait-on améliorer ?

### Analyzer

Maintenant faisons une recherche de type `match` pour 'Star Trek':

```bash
GET name-movies/_search
{
    "query": {
        "match": {
            "title": "Star Trek"
        }
    },
    "size": 50
}
```

Les premiers résultats ressortent bien les films Star Trek mais nous avons aussi des résultats pour Star Wars. Pourquoi? La match query utilise l'analyzer par défaut et va découper en tokens. Donc il renverra tous les films qui ont dans leur titre "Star" ou "Trek" et un score sera calculé pour les trier en fonction du nombre d'apparition de chaque terme.

Maintenant, cherchons tous les films dont le genre et 'sci' avec cette requette:

```bash
GET name-movies/_search
{
    "query": {
        "match_phrase": {
            "genres": "sci"
        }
    }
}
```

Vous allez voir que les résultats peuvent ne pas être ce qu'on attend puisque tous les films avec 'sci' comme 'Sci-Fi' ressortent. Le résulat attendu serait plutôt 0, car aucun film n'a de genre exactement nommé 'sci'.

> Avec le mapping actuel, comment pouvons-nous changer la query pour obtenir le résultat décri ci-dessus (toujours en utilisant le match_phrase) ?

### Modification du mapping

> **Exercice**: Essayons maintenant d'avoir un mapping qui corresponde un peu plus aux recherches que nous souhaitons faire avec les bon types.
>
> - Pour tous les champs texte qui devraient être retournés uniquement en cas d'exact match, définissez un type `keyword`
> - Pour tous les champs texte qui devraient être retournés en cas de match partiel basé sur la pertinence, alors définissez un type `text` avec l'analyzer `english`
> - Les champs numeric par defaut sont des `long`, est-ce ce que nous voulons pour tous nos champs ?
> - Le champ plot doit être uniquement pour la recherche en full-text pas de keyword, et mettre `"fielddata": true`
>
> **Modifiez le mapping pour appliquer l'ensemble des changements.**

## La recherche avec ElasticSearch

### La recherche "light"

La syntaxe simplifiée, qui permet de debugger rapidement ou tester, et qui est exécutable dans un navigateur:

```text
http://localhost:9200/name-movies/movie/_search?q=title:star
http://localhost:9200/name-movies/movie/_search?q=title:trek&explain=true
```

Pour plus de détails: [Elasticsearch URI Search Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-uri-request.html)

**Note**: ne pas utiliser en production cette syntaxe. C'est facile à casser, détourner, etc...

## Queries & Filters

Deux types de contexte existent pour rechercher du contenu:

- query: répond à la question _A quel point ce document correspond à cette clause ?_ En plus de savoir si le document répond ou pas à la clause, un `_score` est calculé, il représente la pertinence du document par rapport à cette clause.
- filter: répond à la question _Ce document répond-il à cette clause ?_ La réponse est un simple oui ou non, il n'y a pas de score calculé.

Les filtres souvent utilisés vont être automatiquement mis en cache par ElasticSearch pour améliorer les performances.

Exemple:

```bash
GET name-movies/_search
{
    "query": {
        "bool": {
            "must": { "term": { "title": "trek" } },
            "filter": { "range": { "year": { "gte": 2010 } }}
        }
    }
}
```

Types de filtres:

- `term` filtre par la valeur exact `{ "term": { "year": 2014 } }`
- `terms` match si une des valeurs dans la liste match `{"terms": { "genres": ["Sci-Fi", "Adventure"] }}`
- `range` trouve un nombre ou un date entre deux bornes `{ "range": { "year": { "gte": 2010 } }`
- `exists` trouve les documents où le champ existe `{ "exists": { "field": "tags" } }`
- `bool` combine les filtres avec la logique booléenne (`must` (AND), `must_not` (NOT), `should` (OR))

Type de queries:

- `match` fait une recherche avec les analyzers (recherche full-text) `{ "match": { "title": "The Force Awakens" } }`
- `multi_match` fait la même recherche mais sur plusieurs champs `{ "multi_match": { "query": "star", "fields": ["title", "plot"]`}}
- `bool` même fonctionnalité que pour les filtres mais le résultat possède un score pour la pertinence

## Proximité

Certaines recherches full-text permettent d'ajouter un paramètre de proximité pour faire des recherches de type "Fuzzy". Cela utilise la comparaison de texte avec la [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance).

Prenons l'exemple suivant. Je veux rerchercher tous les films qui contiennent "Star"

```bash
GET name-movies/_search
{
    "query": {
        "match": {
            "title": "Star"
        }
    }
}
```

Nous obtenons 26 films. Oui mais voilà, quand on tape sur google rapidement, nous pouvons tapper `Stra` au lieu de `Star`. Et là nous n'obtenons plus de résultat.

```bash
GET name-movies/_search
{
    "query": {
        "match": {
            "title": "Stra"
        }
    }
}
```

Du coup on change un peu la structure de la query pour ajouter la `fuzziness`

```bash
GET name-movies/_search
{
    "query": {
        "match": {
            "title": {
              "query": "Stra",
              "fuzziness": "auto"
            }
        }
    }
}
```

Et là nous obtenons 28 films, les 26 de "Star" plus 2 nommés "Straw Dogs". En effet la distance de Levenshtein est la même entre "Straw" "Star" et "Stra".
Plus de détails sur la [fuzziness](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#fuzziness).

### Match partiel

Deux type de query notemment permettent une recherche avec un résultat partiel:

- `prefix` qui comme son nom l'indique va permettre de rechercher avec un "commence par"

```bash
GET name-movies/_search
{
    "query": {
        "prefix": {
            "title.keyword": "Star"
        }
    }
}
```

- `wildcard`

```bash
GET name-movies/_search
{
    "query": {
        "wildcard": {
            "title.keyword": "Star *"
        }
    }
}
```

### Pagination

On définit l'index de document de départ avec le mot clé `from` et le nombre de documents à retourner avec `size`.

```bash
GET name-movies/_search
{
    "from": 2,
    "size": 2,
    "query": {
        "match": {
            "title": "Star"
        }
    }
}
```

### Tri

Comme en SQL, on peut définir la manière de trier les résultats. Par défaut le score calculé (la pertinence) est utilisé.

**Attention**: il n'est pas possible de trier sur un champ de type full-text

```bash
GET name-movies/_search
{
    "query": {
        "match": {
            "title": "Star"
        }
    },
    "sort": [
        {
            "year": "desc"
        }
    ]
}
```

> **Exercice**: Effectuer un tri descendant sur les titres (c'est possible sans changer le mapping).

### Exercices de recherche

> Effectuez les recherches suivantes, pour chacune d'elles, noter la query et le nombre de résultat:
>
> 1. Donner la liste des films dont le titre contient "Star Wars" (requête de type match)
>
> 2. Films ’Star Wars’ dont le réalisateur (directors) est ’George Lucas’ (requête booléenne)
>
> 3. Films dans lesquels ’Harrison Ford’ a joué
>
> 4. Films dans lesquels ’Harrison Ford’ a joué dont le résumé (plot) contient ’Jones’
>
> 5. Films dans lesquels ’Harrison Ford’ a joué dont le résumé (plot) contient ’Jones’ mais sans le mot ’Nazis’
>
> 6. Films de ’James Cameron’ dont le rang devrait être inférieur à 1000 (boolean + range query)
>
> 7. Films de ’James Cameron’ dont le rang doit être supérieur à 5, sans être un film d’action ni une romance
>
> 8. Films de ’J.J. Abrams’ sorties (released) entre 2010 et 2015 (boolean query avec filter)

## Aggrégations

### Counts & Averages

C'est très similaire au SQL. Prenons pour exemple l'aggrégation par notes de films:

```bash
GET name-movies/_search
{
    "aggs": {
        "ratings": {
            "terms": {
                "field": "rating"
            }
        }
    },
    "size": 0
}
```

Vous remarquez deux choses:

- une `size` est spécifiée sinon, tous les hits correspondant à la query (donc ici tous les films étant donné qu'il n'y a pas de query) seront retournés dans le résultat. Nous ne voulons que le résultat de l'aggrégat.
- dans le résultat, la `key` pert la précision dans le cas d'un float et on obtient 6.999999 à la place de 6.7. Pour des terms aggregation il vaut mieux avoir des types `double`

Voici un autre exemple illustrant l'équivalent d'un `WHERE` + `GROUP BY` en ne gardant que les films qui ont une note de 5.0:

```bash
GET name-movies/_search
{
    "query": {
        "match": { "rating": 5.0 }
    },
    "aggs": {
        "ratings": {
            "terms": {
                "field": "rating"
            }
        }
    },
    "size": 0
}
```

Enfin, dernier exemple, la note moyenne des films Star Wars:

```bash
GET name-movies/_search
{
    "query": {
        "match_phrase": { "title": "Star Wars" }
    },
    "aggs": {
        "avg_ratings": {
            "avg": {
                "field": "rating"
            }
        }
    },
    "size": 0
}
```

### Histograms & Buckets

Elasticsearch est capable de grouper les données dans des 'buckets' qui peut être utilisé facilement pour créer des histogrammes.

Par exemple, si nous voulons récupérer et grouper toutes les notes par 1.x, 2.x, etc... on peut lancer la requête suivante:

```bash
GET name-movies/_search
{
    "aggs" : {
        "whole_ratings": {
            "histogram": {
                "field": "rating",
                "interval": 1.0
            }
        }
    },
    "size": 0
}
```

Ou encore, combien de films il y a par décennies:

```bash
GET name-movies/_search
{
    "aggs" : {
        "release": {
            "histogram": {
                "field": "year",
                "interval": 10
            }
        }
    },
    "size": 0
}
```

### Subaggregation

Il est possible d'imbriquer les aggrégats. Par exemple, récupérons la moyenne des films pour les réalisateurs que l'on souhaite:

```bash
GET name-movies/_search
{
    "query": {
        "terms": {
            "directors.keyword": ["J.J. Abrams","James Cameron","Steven Spielberg"]
        }
    },
    "aggs": {
        "directors": {
            "terms": {
                "field": "directors.keyword"
            },
            "aggs": {
                "avg_rating": { "avg": { "field": "rating" } }
            }
        }
    },
    "size": 0
}
```

### Exercices

Pour vous aider dans les exercices, voilà la documentation pour les [aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/6.x/search-aggregations.html).

> Effectuer les aggrégations suivantes, pour chacune d'elles, noter la query et le résultat:
>
> 1. Donner pour les années 2000, 2001 et 2002 le nombre de film sortis
>
> 2. Par catégorie de film, donner leurs occurences
>
> 3. Note moyenne de toute la base de films
>
> 4. Note moyenne des films de George Lucas
>
> 5. Note moyenne des films par genre
>
> 6. Notes minimum, maximum et moyenne des films par genre
>
> 7. Top 3 des films les mieux notés par décennie pour les films d'action réalisé entre 1980 et 2010
>
> 8. Compter le nombre de films pour les tranches de note 0-2.5, 2.5-4, 4-6, 6-7.5, 7.5-10 (en une seule recherche)
>
> 9. Termes les plus utilisés (agrégat : significant_terms) dans les descriptions (plot) des films de George Lucas
>
> 10. Liste distinct des réalisateurs de films d’aventure, qui ont au moins 5 films, ordonnée par note moyenne des films.
