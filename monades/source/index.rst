Les monades, ça fait peur. On en entend généralement parler quand on se lance dans l’étude du Haskell, dont elles sont souvent présentées comme la principale difficulté, voire de la programmation fonctionnelle de manière générale. Pourtant, si l’on se restreint à l’essentiel, à savoir ce que c’est, et quelle est leur utilité, c’est vraiment tout bête.

Alors voilà l’objectif que se fixe ce cours : faire comprendre simplement ce que sont les monades, et guère plus. On n’expliquera pas comment créer une monade dans les formes, on ne parlera pas des lois des monades, *a fortiori* des mathématiques derrière les monades, on ne parlera pas non plus des usages avancés des monades, comme les transformateurs de monades ou les monades additives.

Juste l’essentiel, l’indispensable. Qu’est-ce que c’est. Quelques exemples de cas où elles sont utiles. Quelques exemples de cas où elles *ne sont pas* utiles. Le reste sera traité dans d’autres cours.

Tous les codes sont expliqués : il n’est donc pas nécessaire de connaître le Haskell pour comprendre ce cours. Malgré tout, en connaître les principales syntaxes aide grandement. La troisième section montre des exemples de monades dans d’autres langages que le Haskell, mais le code est là aussi expliqué, et ne nécessite pas de connaître le langage employé.

En revanche, le cours considère comme acquis les concepts usuels de la programmation (fonction, variable, etc.) et sera difficilement [#]_ accessible à quelqu’un qui n’aurait aucune pratique de celle-ci.

.. |br| raw:: html

   <br />

.. note::

    **Prérequis** |br|
    Vocabulaire usuel de la programmation. |br|
    Avoir déjà résolu des problèmes concrets au moyen d’un programme.

    **Prérequis optionnel** |br|
    Niveau moyen en `Haskell`__\ .

    **Objectifs** |br|
    Comprendre ce qu’est une monade. |br|
    Savoir reconnaître une monade quand on en voit une. |br|
    Savoir dans les grandes lignes quand utiliser une monade.

    **Aspects non traités** |br|
    La classe de type ``Monad`` en Haskell. |br|
    Les lois des monades. |br|
    Les monades additives et les transformateurs des monades. |br|
    Les limites des monades et comment les dépasser. |br|
    Les mathématiques derrière les monades.

.. __: http://lyah.haskell.fr

.. contents::

Définition et exemples
======================

.. question::

    Une monade, *qu’es acò* ?

Une monade, c’est une boite, pouvant contenir un ou plusieurs objets de même nature, conçue de telle manière que l’on puisse placer un/des objet(s) dans la boite, et par ailleurs, sortir le contenu de la boite juste le temps d’effectuer une action sur lui, avant de le remettre dans la boite.

Notez bien que les objets sont de même nature *à un instant donné* : l’action effectuée sur les objets peut en changer la nature, du moment qu’elle change la nature de *tous* les objets. Pour prendre une comparaison concrète, on peut sortir des sacs de grain et remettre des sacs de farine.

Voilà pour l’approche intuitive. De manière un peu plus concrète, et avec du vocabulaire plus formel, une abstraction informatique peut être considérée comme une monade si elle réunit les éléments suivants.

- Un type qui permet d’encapsuler un ou plusieurs objets **de même type**, c’est-à-dire un type paramétré [#]_. C’est le terme en usage en Haskell ou en OCaml, mais d’autres langages utilisent des noms différents : en Rust, on parlera des génériques, en C++, d’une *template class*, etc. Notez bien que la définition n’autorise que les types à un seul paramètre.
- Une fonction qui prend un objet de tout type (avec éventuellement certaines limites) et l’encapsule dans le type précité [#]_, créant ainsi une **valeur monadique** [#]_. Selon les langages, elle est appelée ``return``, ``unit`` ou encore ``lift``.
- Une fonction qui prend en paramètres une valeur monadique (c’est-à-dire un objet du type utilisé par la monade contenant un ou plusieurs objets) et une autre fonction — laquelle prend en paramètre un objet du type encapsulé et renvoie une nouvelle valeur monadique — puis applique ladite fonction à la valeur encapsulée dans le premier paramètre [#]_ [#]_. C’est un peu aride, dit comme cela, mais un exemple viendra vite éclairer les choses. Cette fonction est habituellement appelée ``bind``.

Prenons le Haskell comme langage de travail, et essayons de créer la plus simple des monades. Il nous faut d’abord un type paramétré. Comme on veut créer la plus simple de toutes les monades, ce type paramétré ne pourra contenir qu’un seul objet.

.. code:: haskell

    data MonType a = MonConstructeur a

Pour ceux qui ne seraient pas familiers avec le langage, le mot-clé ``data`` signifie que l’on va définir un nouveau type. ``MonType`` est le nom du type, et le ``a`` signale que c’est un type paramétré à un seul paramètre. Ce qui vient après le ``=`` est un constructeur de valeur, là aussi à un seul paramètre : on lui fournit une valeur de n’importe quel type, et il l’emballe pour en faire un objet de type ``MonType``.

Passons à la fonction ``return``. *Stricto sensu*, le constructeur de valeur dont nous venons de parler fait déjà le travail, mais on va la définir quand même, juste pour être complets.

.. code:: haskell

    return :: a -> MonType a
    return x = MonConstructeur x

La première ligne n’est pas entièrement nécessaire, elle se contente de dire que ``return`` prend un objet de n’importe quel type, et renvoie une valeur monadique de type ``MonType`` contenant ce même type. La seconde ligne dit strictement ce que nous avons fait remarquer trois lignes plus haut : ``return`` est parfaitement équivalent au constructeur de valeur.

Vient la partie la plus complexe, à savoir ``bind``. En voici le code.

.. code:: haskell

    bind :: MonType a -> (a -> MonType b) -> MonType b
    bind (MonConstructeur objet) fonction = fonction objet

Là encore, la première ligne n’est pas totalement nécessaire. Elle fixe les mêmes conditions de type que données plus haut dans l’explication formelle. Quant à la seconde, elle est un tout petit peu plus complexe, alors voyons les choses pas à pas.

La fonction ``bind`` prend deux paramètres, ``(MonConstructeur objet)`` et ``fonction``. Le premier est rédigé de manière à utiliser le filtrage par motif, qui permet d’accéder de manière élégante à la valeur contenue dans l’emballage ``MonType``. La partie après le ``=`` définit le fonctionnement de ``bind``, à savoir simplement appliquer ``fonction`` au contenu de la valeur monadique.

Son utilisation est alors très simple.

.. code:: haskell

    multiplier_par_deux :: Int -> MonType Int
    multiplier_par_deux x = return (x * 2)

    valeur_monadique = return 4
    nouvelle_valeur_monadique = bind valeur_monadique multiplier_par_deux

La fonction ``nouvelle_valeur_monadique`` renverra ``8`` encapsulé dans un objet de type ``MonType``. Et voilà ! Vous avez devant vous votre première monade. Moins effrayant qu’il n’y paraît, n’est-ce pas ? Et parfaitement inutile, soyons honnêtes. Mais elle permet de mettre en lumière une vérité première sur les monades, qu’il faut toujours avoir à l’esprit quand on s’intéresse à elles.

.. note::

    Une monade n’est pas un concept mystique, c’est un outil, plus ou moins utile selon les situations, et répondant à un cahier des charges précis.

.. question::

    Le cahier des charges reste très abstrait.

Alors voyons quelques exemples et contre-exemples, toujours en Haskell.

Le type ``Maybe`` peut être défini comme suit, accompagné de ses fonctions ``return`` et ``bind``.

.. code:: haskell

    data Maybe a = Nothing | Just a

    return :: a -> Maybe a
    return x = Just x

    bind :: Maybe a -> (a -> Maybe b) -> Maybe b
    bind (Just a) f = f a
    bind Nothing _ = Nothing

La situation est très légèrement plus complexe, mais vous devriez reconnaître la structure générale. On commence par définir le type paramétré ``Maybe``, qui peut prendre deux formes différentes : soit ``Nothing``, qui n’encapsule rien, soit ``Just`` qui, lui, peut encapsuler n’importe quoi.

Comme pour ``MonType``, la fonction ``return`` est parfaitement équivalente au constructeur de valeur ``Just``, puisque celui-ci se contente de prendre une valeur, et de l’emballer dans un objet de type ``Maybe``.

La fonction ``bind`` est plus nettement différente, mais vous remarquerez que la définition de son type suit strictement le même modèle que ``MonType``. Que disent les deux lignes restantes ? Que si la valeur monadique est ``Nothing``, c’est-à-dire n’encapsule rien, alors on ne peut pas appliquer une fonction sur une valeur qui n’existe pas, et on se contente de renvoyer à nouveau ``Nothing``. Si au contraire la valeur monadique a la forme ``Just quelque_chose``, alors on applique la fonction sur ce quelque chose, comme prévu.

Comme vous le voyez, le triplet ``Maybe a``, ``return``, ``bind`` respecte les conditions imposées, et constitue donc une monade. Quelle est son utilité ? Elle sert à gérer les opérations qui peuvent échouer. Mais nous reviendrons plus tard là-dessus.

Il existe un autre type dans la bibliothèque standard du Haskell, qui sert à gérer les opérations qui peuvent échouer, et il est défini comme suit.

.. code:: haskell

    data Either a b = Left a | Right b

On voit d’emblée que ce type ne peut pas servir de base à une monade. En effet, il possède *deux* paramètres, et non un seul, ce qui le disqualifie d’office. Vous retiendrez donc que c’est la forme, et non l’utilité, qui fait la monade.

On s’en convaincra en voyant cette autre monade, assez nettement différente de ``Maybe``.

.. code:: haskell

    data Liste a = ListeVide | ListePasVide a (Liste a)
    -- ListePasVide encapsule le premier élément de la liste,
    --   puis le reste de la liste d’un bloc.

    return :: a -> Liste a
    return x = ListePasVide x ListeVide
    -- Une liste à un seul élément contient donc un premier élément,
    --   puis un « reste de la liste » qui est vide.

    bind :: Liste a -> (a -> Liste b) -> Liste b
    bind ListeVide _ = ListeVide
    bind liste f = concat (map f liste)

Il ne s’agit pas du vrai type de liste du Haskell, car celui-ci utilise une syntaxe trop particulière pour rester compréhensible, mais le principe est le même. On a un type à un seul paramètre, qui est cette fois récursif : il encapsule, outre une valeur, un autre objet du même type que lui. En pratique, cela revient à encapsuler plusieurs objets du même type.

La fonction ``return`` se contente d’encapsuler la valeur fournie en entrée dans une liste dont elle est le seul élément. Quant à ``bind`` elle applique la fonction ``f`` sur chaque élément de la liste fournie en entrée, avant « d’écraser » les deux niveaux de liste en un seul. Cet exemple vous permettra d’y voir un peu plus clair.

.. code:: haskell

    ma_fonction :: Int -> [Int]
    ma_fonction x = [x + 1, x + 10]
    -- ``ma_fonction`` prend un entier et renvoie une liste composée de :
    --  — cet entier plus 1 ;
    --  — cet entier plus 10.

    ma_liste :: [Int]
    ma_liste = [0, 12, 54]
    -- Une simple liste d’entiers.

    bindons :: [Int]
    bindons = bind ma_liste ma_fonction
    -- Donne pour résultat ``[1, 10, 13, 22, 55, 64]``. Comment en arrive-t-on là ?
    -- Pour commencer, ``map ma_fonction ma_liste`` applique ``ma_fonction`` à chaque élément
    --   de ``ma_liste``. On obtient donc le résultat ``[[1, 10], [13, 22], [55, 64]]``.
    -- Puis ``concat`` « écrase » les deux niveaux de liste en un seul, donnant le résultat
    --   attendu. On a bien une liste d’entiers en entrée, et une liste d’entiers en sortie.

Le détail du fonctionnement importe peu. Ce qu’il faut retenir, c’est que la fonction ``bind`` respecte strictement les conditions imposées par le cahier des charges, même s’il lui faut biaiser un peu pour cela, et cela suffit à faire une monade du type liste accompagné de ces deux fonctions.

Utilité
=======

.. question::

    Les monades sont un outil, certes, mais à quoi sert cet outil ?

Il n’y a qu’une seule réponse possible à cette question : cela dépend de la monade. Nous avons vu dans la section précédente qu’une monade pouvait tout simplement ne servir à rien. Mais supposons que nous ne conservions que les « bonnes » monades, celles qui sont utiles au programmeur ?

L’utilité la plus évidente est de simplifier le code utilisant le type qui sert de base à la monade. Revenons à notre type ``Maybe`` : on peut l’utiliser pour écrire une fonction de division un peu plus propre que celle de départ. Comparez en effet ces deux fonctions.

.. code:: haskell

    divMoche :: Double -> Double -> Double
    divMoche x y = if y == 0
                   then error "Division par 0 interdite !"
                   else x / y

    divM :: Double -> Double -> Maybe Double
    divM x y = if y == 0 then Nothing else Just (x / y)

Voyez la très légère différence dans la déclaration de type de chaque fonction. À quoi cela a-t-il servi ? La fonction ``error`` abandonne l’exécution du programme et affiche une erreur. Si votre programme a pour seule utilité de faire une division, ce n’est pas très grave, mais si votre calculette plantait définitivement chaque fois que vous cherchez à faire une opération n’ayant pas de sens, cela deviendrait vite pénible.

Alors qu’avec ``divM``, c’est le résultat lui-même qui signale qu’il y a eu une erreur. Supposons alors que nous voulions faire plusieurs divisions à la suite [#]_. Il faut, à chaque division, vérifier que la précédente n’a pas abouti à une erreur. Cela donnerait le code qui suit, pour trois divisions successives.

.. code:: haskell

    divM_3 :: Double -> Double -> Double -> Double -> Maybe Double
    divM_3 x y1 y2 y3 =
        case divM x y1 of
            Nothing -> Nothing
            Just x2 -> case divM x2 y2 of
                Nothing -> Nothing
                Just x3 -> divM x3 y3

Pour ceux qui ne connaîtraient pas la syntaxe, ``case <qqch> of`` compare la valeur de ``<qqch>`` aux valeurs ou motifs situés avant les flèches, et exécute le traitement correspondant situé après la flèche. C’est une forme étendue de ``if … then … else``. Et pour ajouter une nouvelle division, il faudrait ajouter encore un niveau d’imbrication. Cela n’est évidemment pas optimal, et c’est là qu’intervient la monade.

Reprenons notre fonction ``divM`` de plus haut, ainsi que les fonctions ``return`` et ``bind`` que nous avions écrites dans la première section. Voici comment on pourrait écrire ``divM_3``.

.. code:: haskell

    divM_3 :: Double -> Double -> Double -> Double -> Maybe Double
    divM_3 x y1 y2 y3 = return x 
                        `bind` \x1 -> divM x1 y1 
                        `bind` \x2 -> divM x2 y2 
                        `bind` \x3 -> divM x3 y3

Il y a quelques nouveaux éléments de syntaxe à expliciter. Le fait de mettre la fonction ``bind`` entre apostrophes inversées \` permet de placer son premier paramètre avant elle, et son second paramètre après, à la manière d’un opérateur (comme ``+`` ou ``/``).

En outre, la structure ``\x1 -> divM x1 y1`` est une **lambda** ou **fonction anonyme** : elle prend un seul paramètre, ``x1``, et renvoie ce qui se trouve après la flèche. Cela permet de créer à peu de frais des fonctions « diviser par y1 », « diviser par y2 », etc. Lesquelles ne prenant qu’un seul paramètre, peuvent être ``bind``\ ées au résultat de l’opération précédente.

Comme vous le voyez, une fois que l’on est habitué à la syntaxe, cette nouvelle version est beaucoup moins pénible à écrire, à comprendre, et à modifier si l’on veut y rajouter des divisions successives. Le Haskell [#]_ a même poussé les choses plus loin : lorsque la monade est définie selon certaines règles (que nous n’aborderons pas dans cette introduction), on peut écrire les choses ainsi.

.. code:: haskell

    divM_3 :: Double -> Double -> Double -> Double -> Maybe Double
    divM_3 x y1 y2 y3 = do
        x1 <- return x
        x2 <- divM x1 y1
        x3 <- divM x2 y2
        divM x3 y3

On s’éloigne du fonctionnement réel des opérations, mais le sens de l’ensemble apparaît plus clairement. Mais ce n’est pas le seul intérêt d’une monade, et pour ainsi dire, ce serait même le plus marginal.

.. question::

    Tant mieux, parce que ça paraît très compliqué pour pas grand chose, pour l’instant.

Une monade a surtout l’avantage de créer un **contexte** de programmation. Concrètement, qu’est-ce que cela signifie ? Deux choses. D’un point de vue formel, utiliser une séquence de ``bind`` permet de s’assurer que le traitement ne sorte jamais de la monade.

Si vous vous souvenez de la définition intuitive donnée au début de ce cours, une monade est une boite, dans laquelle on met des données, pour ne plus les en ressortir. En pratique, ``bind`` ressort les données de la boite, mais la fonction passée en paramètre de ``bind`` les y remet aussitôt : du point de vue du programmeur, tout se passe en coulisse.

Du point de vue du fond, réaliser un traitement au sein d’une monade donne un sens à ce traitement, sens qui dépend — précisément — de la monade employée. Par exemple, notre monade ``Maybe`` signifie « **Attention, ce morceau de programme peut échouer !** ».

La monade de liste, que nous avons vue dans la première section, sert quant à elle à signaler un traitement dont chaque étape peut avoir zéro, un, ou plusieurs résultats différents, sans qu’on puisse vraiment en prévoir le nombre exact.

Par exemple, une intelligence artificielle pour jeu de Puissance 4, qui cherche à déterminer toutes les configurations possibles sur les six coups à venir, pour trouver la plus favorable : certaines branches mèneront à des culs de sac, d’autres offriront plus ou moins de possibilités de poursuivre, certaines colonnes pouvant se trouver pleines.

.. question::

    Et la célèbre monade ``IO`` dans tout ça ?

C’est certainement la plus importante de toutes. Une fois entré dans cette monade, il est absolument impossible d’en sortir, et elle confère le sens de « **Attention ! Attention ! Comportement impur ici, on modifie le monde réel. Effets de bord à prévoir.** ». C’est le moyen qu’a trouvé le Haskell de représenter avec des outils de programmation fonctionnelle pure des opérations qui génèrent des effets de bord, comme afficher un texte à l’écran ou lire le contenu d’un fichier.

Voyons cela plus en détail. Les opérations d’entrée-sortie représentent deux sévères épines dans le pied de la programmation fonctionnelle pure, et l’utilisation d’une monade permet de contourner les deux problèmes.

Tout d’abord, l’absence d’effet de bord (également appelée **transparence du référentiel**) a pour conséquence qu’une fonction appelée deux fois avec les mêmes paramètres donnera nécessairement deux fois le même résultat. Cela autorise donc à garder ce résultat en mémoire, plutôt que de répéter le traitement lors du deuxième appel : on nomme cela la **mémoïsation**.

Seulement, *quid* de la fonction ``getLine``, qui lit une ligne de texte entrée par l’utilisateur ? Appelée deux fois de suite, la probabilité que l’utilisateur tape strictement la même chose la deuxième fois est infime. Mais comment savoir que cette fonction ne doit pas être mémoïsée, contrairement à toutes les autres ? En la « marquant » du sceau de l’impureté, ce qui se fait en lui donnant le type ``IO String`` plutôt que ``String``.

Plus fourbe : la transparence du référentiel a aussi pour conséquence que si deux fonctions ne dépendent pas l’une de l’autre, l’ordre dans lequel elles sont exécutées n’a strictement aucune importance. Mais il en va tout autrement des opérations d’entrée-sortie : si vous demandez à votre utilisateur de vous fournir son nom puis son prénom, il est indispensable que vous soyez certain de l’ordre dans lequel chacun sera demandé.

En ``bind``\ ant les fonctions au sein de la monade IO, vous créez une dépendance artificielle entre elles, qui oblige celle que vous placez en premier dans le code à être effectivement exécutée la première. Cela peut apparaître plus clairement sur un code.

.. code:: haskell

    prénom = getLine
    nom = getLine
    main = putStrLn ("Bonjour " ++ prénom ++ " " ++ nom)

.. code:: haskell

    main = getLine
        `bind` \prénom -> (getLine
        `bind` \nom -> putStrLn ("Bonjour " ++ prénom ++ " " ++ nom))

Dans le premier code, ``prénom`` et ``nom`` ne dépendent pas l’une de l’autre : rien ne vous permet de savoir si votre programme va commencer par demander le prénom ou le nom [#]_, donc si les informations fournies par l’utilisateur seront dans le bon ordre lorsque le programme les affichera à l’écran.

Dans le second, au contraire, il est indispensable que le premier ``getLine`` ait été exécuté avant que le programme puisse passer à l’exécution de l’ensemble ``\prénom -> […] nom))``, donc au second ``getLine``. On a réussi à forcer un ordre d’exécution, censément incompatible avec la programmation fonctionnelle pure.

Un outil multi-langages
=======================

.. question::

    Les monades sont-elles réservées au Haskell ou aux langages proches de celui-ci ?

Absolument pas ! Il est possible de créer des monades dans de nombreux langages, avec plus ou moins de difficultés. Prenons par exemple le OCaml, un autre des principaux langages fonctionnels, où les monades ne sont pourtant pas d’usage aussi courant qu’en Haskell. La monade ``Maybe`` s’implémenterait simplement comme suit [#]_.

.. code:: ocaml

    type 'a maybe = Nothing | Just of 'a

    let return = Just

    let (bind) m f =
        match m with
        | Nothing -> Nothing
        | Just a -> f a

Contrairement au Haskell, le(s) paramètre(s) des types se place(nt) avant le nom de ceux-ci, d’où la syntaxe ``'a maybe``. Par ailleurs, la structure ``match … with`` sert à faire du filtrage par motif, comme on l’a déjà vu. Et comme vous pouvez le constater, cela reste très facile à mettre en place.

On peut ainsi réaliser des divisions successives sur un nombre, en s’assurant qu’une division par zéro renvoie une erreur, et que celle-ci soit répercutée sur la suite des opérations. Voici un exemple de comment cela pourrait se faire.

.. code:: ocaml

    let divM x y = if y = 0. then Nothing else Just (x /. y)

    let impossible = Just 35.
                     bind fun x -> divM x 5.
                     bind fun x -> divM x 0.
                     bind fun x -> divM x 7.

Si l’on peut placer la fonction ``bind`` entre ses deux arguments, comme on le fait en Haskell, c’est parce qu’on l’a déclarée entourée de parenthèses dans le code un peu plus haut. Sans cela, la syntaxe serait très lourde.

Autre exemple, le Rust : un langage multi-paradigmes, qui intègre plutôt bien les outils de la programmation fonctionnelle. Voici comment on pourrait implémenter la monade ``Maybe`` [#]_.

.. code:: rust

    enum Maybe<A> {
        Just(A),
        Nothing
    }

    fn lift<A>(a : A) -> Maybe<A> {
        Maybe::Just(a)
    }

    impl<A> Maybe<A> {
        fn bind<B, F>(self, f : F) -> Maybe<B>
            where F : Fn(A) -> Maybe<B>    {
            match self {
                Maybe::Nothing => Maybe::Nothing,
                Maybe::Just(a) => f(a)
            }
        }
    }

Quelques explications pour ne pas être bloqués par la syntaxe.

- Les notations entre chevrons, par exemple ``<A>``, servent à définir les types et fonctions paramétrées.
- On ne peut appeler une fonction ``return``, car c’est un mot-clef du langage, aussi, on se rabat sur ``lift``.
- Chaque argument d’une fonction doit être typé explicitement, avec la syntaxe ``: <type>``, et le type de retour de la fonction est spécifié après la flèche ``->``.
- La syntaxe ``impl<A> Maybe<A> { … }`` fait de ``bind`` une méthode du type ``Maybe``. Ce n’est pas absolument indispensable, mais cela améliore considérablement la syntaxe de l’appel.
- Toute la partie avec ``F`` et la ligne qui commence par ``where`` est nécessaire pour pouvoir passer des fonctions anonymes en paramètre de ``bind``, il n’est pas nécessaire de la comprendre dans le détail.
- La structure ``match`` est le moyen utilisé par le Rust pour implémenter du filtrage par motif, comme en OCaml.

On reprendra l’exemple des divisions successives pour montrer la mise en action de cette monade.

.. code:: rust

    fn divM(x : f64, y : f64) -> Maybe<f64> {
        if y == 0.0 { return Maybe::Nothing; }
        lift(x / y)
    }

    fn main()   {
        let m = Maybe::Just(35.0).bind(|x| divM(x, 5.0))
                    .bind(|x| divM(x, 0.0))
                    .bind(|x| divM(x, 7.0));

        match m {
            Maybe::Nothing => println!("Nothing"),
            Maybe::Just(ref a) => println!("Just {}", a)
        }
    }

La syntaxe ``|x| divM(x, 5.0)`` permet de définir une fonction anonyme, ou *closure* comme elles sont appelées par les Rustacés : dans notre cas, elle prend en entrée un argument ``x``, en l’occurrence, l’éventuel contenu de la valeur monadique, et renvoie le résultat de ``divM(x, 5.0)``, qui est bien une valeur monadique.

Comme vous pouvez le voir, le fait d’avoir fait de ``bind`` une méthode de ``Maybe`` permet de chaîner les appels à celle-ci dans une syntaxe fort élégante.

À vrai dire, il n’est même pas indispensable d’utiliser un langage fonctionnel ou intégrant fortement le paradigme fonctionnel, pour pouvoir créer une monade. Voici l’adaptation en C++11 de notre monade ``Maybe``.

.. code:: cpp

    #include <functional>
    #include <utility>

    template <typename A>
    class Maybe {

        private:
        bool _proprio{};
        A const* const _valeur{};

        public:
        constexpr Maybe() = default;
        constexpr Maybe(A const& valeur) noexcept : _valeur{&valeur}    {   }
        constexpr Maybe(A&& valeur)      noexcept
            : _proprio{true}, _valeur{new A{valeur}}
            {   }

        constexpr Maybe(Maybe const& m)  noexcept : _valeur{m._valeur}  {   }
        constexpr Maybe& operator=(Maybe const& m) noexcept {
            _proprio = false;
            _valeur = m._valeur;
            return *this;
        }

        ~Maybe() { if(_proprio) delete _valeur; }

        inline constexpr static Maybe Nothing() noexcept {
            return Maybe();
        }

        template<typename ...Args>
        inline constexpr static Maybe Just(Args&&... args) noexcept {
            return Maybe(std::forward<Args>(args)...);
        }

        template<typename F, typename ...Args>
        inline constexpr Maybe bind(F&& f, Args&&... args) const {
            if (!_valeur) return Nothing();
            return Maybe(f(*_valeur, std::forward<Args>(args)...));
        }

        A valeur() const {
            if (_valeur) return *_valeur;
            return 0;
        }
    };

C’est en voyant combien le code est beaucoup plus encombré que l’on comprend que les monades ne sont pas très naturelles dans ce langage. Il me serait impossible de totalement détailler le code, alors je m’en tiendrai à signaler que ``template <typename A>`` sert à gérer les types paramétrés.

Mais une fois ce gros travail effectué en amont, et en supposant toujours notre succession de divisions, on peut alors utiliser la syntaxe suivante.

.. code:: cpp

    int main()  {
        auto divM  = [](auto x, auto y) {
            if (y == 0) return Maybe<double>::Nothing(); 
            return Maybe<double>::Just(x / y);
        };
        auto div0M = [divM](auto x){ return divM(x, 0.0); };
        auto div5M = [divM](auto x){ return divM(x, 5.0); };
        auto div7M = [divM](auto x){ return divM(x, 7.0); };

        Maybe<double>::Just(35.0).bind(div5M).bind(div0M).bind(div7M);

        return 0;
    }

Si la syntaxe finale est assez élégante, il n’en reste pas moins qu’utiliser des fonctions anonymes comme dans les autres langages est une véritable plaie, et que faire une définition « propre » de notre monade ``Maybe`` est excessivement difficile. C’est pourquoi vous rencontrerez assez peu de monades dans des langages impératifs.

Mais cela arrive ! La bibliothèque `ReactiveX`__ de Java, Groovy, Clojure… offre le triplet (``Observable``, ``from``, ``subscribe``) qui constitue une monade de la plus belle espèce.

.. __: https://github.com/ReactiveX/RxJava/wiki/How-To-Use-RxJava

En revanche, s’il est en théorie possible de créer des monades dans un langage utilisant un système de typage dynamique, cela s’avère très artificiel, comme le montre cet exemple en Erlang.

.. code:: erlang

    return (A) -> {just, A}.

    bind ({nothing}, _) -> {nothing};
    bind ({just, A}, F) -> F(A).

    divM (X, Y) when Y == 0 -> {nothing};
    divM (X, Y) when Y /= 0 -> return(X / Y).

    impossible() ->
        bind(
            bind(
                bind(return(35.0), fun (X) -> divM(X, 5.0) end)
            , fun (X) -> divM(X, 0.0) end)
        , fun (X) -> divM(X, 7.0) end).

Le type ``Maybe`` n’est pas défini, puisque cela ne fait pas partie de l’ordre des possibles en Erlang, et il est simulé par un tuple, soit de la forme ``{nothing}``, soit de la forme ``{just, <quelque chose>}``. Dès lors, on peut en effet implémenter la fonction ``bind`` à l’aide d’un simple filtrage par motif, mais il n’est fait aucune vérification sur la nature de la fonction passée en paramètre. Le résultat est une imbrication assez inélégante de ``bind( … )``.

Cela nous amène à la constatation suivante : s’il est effectivement possible de créer des monades dans presque tous les langages, certains d’entre eux se montrent nettement rétifs à cette utilisation. Alors, est-ce un problème ?

Nécessité ?
===========

.. question::

    Est-il indispensable de maîtriser les monades pour utiliser un langage fonctionnel pur ?

Absolument pas. Je vais le mettre en gros, pour être sûr que le message passe bien.

.. important::

    Il n’est pas nécessaire de comprendre les monades pour programmer dans un langage fonctionnel pur : c’est une notion avancée, d’usage limité.

Il y a plusieurs raisons à cela. Pour commencer, en Haskell, la seule monade dont le langage ne peut absolument pas se passer, c’est la monade ``IO``. Or gérer les entrées-sorties au moyen d’une monade n’est qu’une solution parmi d’autres.

On peut bien sûr accepter d’intégrer l’impureté dans le langage, comme en OCaml, ou de faire une croix sur les interactions avec le monde extérieur, comme le langage Charity, mais ce ne sont que des pis-aller. En restant parfaitement pur, deux autres solutions se sont fait jour à l’heure actuelle.

- Les langages Clean, Idris et Mercury intègrent l’**unicité** dans leur système de types. En très simplifié, cela consiste à déclarer qu’une variable donnée est à usage unique, cela étant intégré dans son type, si bien que la vérification du typage échouera si cette variable est employée à deux endroits différents du code : le fichier dans lequel on vient de lire une ligne n’est pas le même, du point de vue du programme, que celui dans lequel on va lire la ligne suivante.
- Le langage Elm utilise une abstraction appelée « signal », dont le fonctionnement serait trop long à expliquer ici : on s’en sert pour marquer qu’un objet peut voir sa valeur évoluer dans le temps, et qu’il faudra alors répéter le traitement avec la nouvelle valeur.

Ensuite, même au sein du Haskell, les monades permettent de résoudre un type très spécifique de problèmes : ceux qui nécessitent de composer des fonctions de manière non triviale. Dans la pratique, vous pourrez écrire de nombreux programmes sans jamais avoir besoin de créer une monade pour arriver à vos fins.

Quant à utiliser les monades déjà présentes dans la bibliothèque standard, vous ne pourrez certainement pas échapper à la monade ``IO``. Pour les autres, en revanche, c’est différent : les listes et le type ``Maybe``, par exemple, sont très utiles en soi, mais les occasions où il est nécessaire d’utiliser leur aspect monadique sont en réalité assez rares.

Reste donc la monade ``IO``. Est-il nécessaire de comprendre comment fonctionnent les monades pour s’en servir ? Non. La notation ``do`` s’abstrait presque complètement du fonctionnement réel de la monade sous-jacente, et offre une syntaxe plus claire à l’usage.

----------

Comme je manque cruellement d’imagination pour représenter une monade d’une manière qui ne fasse pas peur, j’ai lâchement emprunté une illustration tirée de `Learn You a Haskell For Great Good!`__ de Miran Lipovača pour mon logo ; celui-ci reste donc la propriété de M. Lipovača, distribuée sous licence `Creative Commons BY-SA-NC`__\ .

.. __: http://lyah.haskell.fr/
.. __: http://creativecommons.org/licenses/by-nc-sa/3.0/deed.fr

Merci à `Vayel`__ pour ses très nombreux commentaires en bêta, à `gbdivers`__ pour son intervention sur le code C++ et à `Javier`__ pour son exemple de ReactiveX.

.. __: https://zestedesavoir.com/membres/voir/Vayel/
.. __: https://zestedesavoir.com/membres/voir/gbdivers
.. __: https://zestedesavoir.com/membres/voir/Javier

Si vous n’avez pas trouvé votre bonheur avec mon explication, et que vous connaissez un peu d’anglais, vous pourrez trouver sur `cette page`__ des liens vers la quasi-totalité des cours anglophones sur les monades existant à ce jour. Environ 85 à l’heure où j’écris ces lignes. Oui.

.. __: https://wiki.haskell.org/Monad_tutorials_timeline

----------

.. [#] Le mot est faible…

.. [#] C’est la boite de la définition intuitive.
.. [#] On place l’objet dans la boite.
.. [#] La boite et son contenu.
.. [#] On sort le contenu de la boite, on lui applique un traitement, et on le remet dans la boite.
.. [#] Comme dit dans la définition intuitive, cette fonction peut modifier le type de l’objet encapsulé, ce n’est pas gênant. On peut, par exemple, avoir une fonction qui prend un flottant en entrée et renvoie un arrondi entier de celui-ci, encapsulé dans le type de la monade.

.. [#] Ce n’est pas fondamentalement utile, avouons-le. Mais l’exemple est simple.
.. [#] Et seulement lui ou les langages qui en sont dérivés, désolé.
.. [#] Si vous avez l’habitude de la programmation impérative, cela vous paraîtra sans doute totalement contre-intuitif, mais c’est ainsi qu’il en va en programmation fonctionnelle pure : ce n’est pas parce qu’un élément se trouve avant un autre dans l’expression qu’il sera nécessairement évalué en premier.

.. [#] Notez bien que la bibliothèque standard du OCaml propose le type ``Option`` qui n’est autre que ``Maybe`` sous un autre nom, mais j’ai préféré rester cohérent entre les exemples.
.. [#] La bibliothèque standard du Rust propose le même type ``Option`` que celle du OCaml.
