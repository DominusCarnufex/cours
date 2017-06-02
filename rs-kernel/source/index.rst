Coder, ce n’est pas mon métier. Je ne fais pas non plus d’études d’informatique. Et vous savez pourquoi c’est génial ? Parce que je peux me permettre de coder des trucs complètement crétins et inutiles sans que qui que ce soit vienne me rabrouer pour autant. Et ce dont je vais vous parler, c’est précisément un de ces bouts de code-là…

.. important::

    Cet article s’adresse aussi bien à des amateurs de noyau Linux qu’à des amateurs de Rust, et je n’attends pas des premiers qu’ils connaissent grand chose à Rust, ni des seconds qu’ils en sachent long sur les entrailles du noyau Linux. En outre, je vais devoir plonger dans certains aspects avancés de Rust, ainsi qu’aborder les rivages maudits de l’assembleur x86_64, et je m’attends à ce que les deux en ignorent tout.

    C’est pourquoi je vais donner beaucoup d’explications qui pourront vous paraître inutiles. Si c’est le cas, n’hésitez pas vous épargner ces parties.

Rust est censé être un excellent langage pour la programmation système, du fait qu’il peut aller très près de la machine tout en offrant une bonne dose de sécurité et d’expressivité dont C manque cruellement. Et une telle affirmation *se doit* d’être mise à l’épreuve.

Ce que Philipp Opperman a déjà fait en se lançant dans l’écriture d’un véritable système d’exploitation en Rust et en `nous en parlant`__ (en anglais, alors qu’il est Allemand, le salaud !). Mais cela ne constitue pas une mise à l’épreuve *réaliste* : un système d’exploitation qui partirait de rien n’a à peu près aucune chance d’être terminé, *a fortiori* d’être effectivement utilisé sur de vraies machines.

.. __: http://os.phil-opp.com/

Alors l’idée serait plutôt de réécrire des bouts du noyau Linux en Rust, de sorte que le changement puisse être incrémental et qu’il ne soit pas nécessaire de réécrire l’intégralité du système d’exploitation. Et parce qu’il faut bien commencer quelque part, cela semble un choix judicieux de parasiter le système qui gère les appels système, et qui est le point d’entrée dans le noyau depuis l’espace utilisateur.

.. note::

    Tester un noyau, c’est chiant comme la pluie. Alors pour me simplifier la vie, j’ai modifié le code source du Linux Mint que j’ai sur mon portable. C’est un noyau 4.8.17, et vous pouvez en trouver le code source complet sous une forme agréable à utiliser sur `Elixir`__.

.. __: http://elixir.free-electrons.com/linux/v4.8.17/source

Mais avant d’aller plus loin, voyons comment de base fonctionnent les appels système sous Linux.

.. contents::

Appels système : comment qu’on fait ?
=====================================

Pour ceux qui l’ignoreraient, on nomme appel système la manière dont les systèmes d’exploitation à la sauce Unix autorisent les logiciels à faire ce que seul le système d’exploitation devrait pouvoir faire, comme discuter avec le matériel, créer de nouveaux processus ou allouer de la mémoire. La plupart des codeurs n’ont pas à se prendre la tête avec des appel système : ils sont bien cachés dans la libc. Mais quand vous appelez ``printf()`` en C ou ``println!()`` en Rust, par exemple, votre code va *in fine* utiliser l’appel système ``write()``.

Pour expliquer les choses de manière très schématique, le système d’exploitation est seul maître à bord de votre PC. Il décide qui a le droit d’utiliser de la mémoire, du temps de processeur, et d’accéder au matériel. Et cela crée une sorte de bac à sable où les logiciels sont autorisés à tourner, sous la surveillance du système d’exploitation : on l’appelle l’**espace utilisateur** (ou *userspace* dans la langue d’Alexandra Daddario).

De temps à autre, les logiciels de l’espace utilisateur font un appel système. Comment cela s’opère en pratique dépend du processeur. Par exemple, sur les processeurs x86_64, c’est l’instruction ``syscall`` qui déclenche l’appel système. Mais sur x86, c’est ``int 0x80``, et sur ARM et AArch64, c’est ``svc 0``.

Et pour expliquer au système ce qu’il en attend, l’espace utilisateur doit mettre les arguments de l’appel système (exactement comme les arguments d’une fonction) dans de toutes petites zones de mémoire très spéciales que l’on appelle des registres : ils font généralement 32 ou 64 bits de long, et ont des noms comme ``rax`` ou ``rdi`` sur x86_64, ``r0`` ou ``ip`` sur ARM, et ``r0`` ou ``r4`` sur PowerPC. Par exemple, voici le prototype de l’appel système ``write()`` en C.

.. code:: c

    ssize_t write(int fd, const void *buf, size_t count)

Sur un processeur x86_64, ``fd`` va aller dans le registre appelé ``rdi``, ``buf`` dans ``rsi`` et ``count`` dans ``rdx``. De plus, le registre appelé ``rax`` contiendra ``1``, qui est le code pour ``write``, et quand l’appel système aura abouti, le résultat sera lui aussi dans ``rax``.

Et pour ajouter à la difficulté, non seulement les conventions d’appel dépendent de la famille de processeurs, mais également la numérotation des appels système. Par exemple, ``write`` est le numéro ``4`` sur x86, sur ARM, il peut valoir ``4`` ou ``0x900004`` selon l’ABI utilisée, et il vaut ``4004`` sur MIPS32, mais ``5001`` sur MIPS64. Et certains appels système n’existent que pour certaines familles de processeurs.

Mais ensuite, que se passe-t-il lorsque l’instruction qui déclenche l’appel système est appelée ?

Avant tout, le système d’exploitation reprend la main et entre dans ce qu’on appelle l’**espace noyau** (*kernelspace*). Sur certaines architectures comme x86_64, cela provoque un changement de privilèges au niveau du processeur même, mais cela n’est pas très important pour nous juste maintenant. Ensuite, une fonction généraliste chargée de gérer les appels système est appelée, qui fait un certain nombre de vérifications, récupère les arguments dans les registres, et sauve l’état de la mémoire tel que l’a laissé le logiciel appelant.

Après quoi, cette fonction multiplexeuse appelle la fonction effectivement destinée à traiter l’appel système correspondant au nombre dans ``rax``. Ici, ce serait la fonction ``sys_write()`` de ``fs/read_write.c``. Quand le traitement est terminé, le résultat est mis dans le registre de retour (ici, ``rax``), l’état de la mémoire est restauré, quelques vérifications supplémentaires sont faites, et finalement, on est renvoyé dans l’espace utilisateur. Sur x86_64, cela se fait au moyen de l’instruction ``sysret``.

Le fait qu’une bonne partie des informations utiles passe de variables C à des registres assembleur et vice versa, et que la numérotation des appels système dépende du processeur, et que l’on puisse passer en argument des valeurs impossibles du fait que les ``int`` sont un type non spécifique, etc. est évidemment propice aux erreurs.

A contrario, Rust nous donne la possibilité de rendre cela nettement plus sûr, et voici comment on s’y prend.

syscall.rs
==========

Je vais commencer par vous donner le code Rust chargé de gérer les appels système en entier, puis j’expliquerai ce qu’il fait.

.. code:: rust

    #![crate_type = "rlib"]
    #![crate_name = "syscall"]

    #![feature(asm)]

    #![no_std]

    pub enum Syscall    {
        Useless(u32, Option<u32>)
    }

    impl Syscall    {
        #[cfg(feature = "userspace")]
        pub fn call(&mut self)  {
            let pointer = self as *mut Syscall;

            if (pointer as usize) < 1024    {
                // Ça ne devrait pas arriver, mais il faudrait
                // prévoir quelque chose au cas où.
            }

            unsafe { asm_syscall(pointer); }
        }

        #[cfg(feature = "kernel")]
        pub fn handle(&mut self)    {
            match *self {
                Syscall::Useless(data, ref mut ret) => *ret = Some(data + 15),
            };
        }
    }

    #[cfg(all(feature = "kernel", target_arch = "x86_64"))]
    #[no_mangle]
    pub unsafe extern fn rust_syscall_handle() {
        let pointer : *mut Syscall;

        asm!(""
           : "={rax}"(pointer)
         : : "memory"
           : "intel", "volatile"
        );

        (*pointer).handle();
    }

    #[cfg(all(feature = "userspace", target_arch = "x86_64"))]
    #[inline(always)]
    pub unsafe fn asm_syscall(ptr : *mut Syscall)   {
        asm!("syscall"
         : : "{rax}"(ptr)
           : "rcx", "r11", "memory"
           : "intel", "volatile"
        );
    }

Avant tout, un brin de configuration.

.. code:: rust

    #![feature(asm)]

Notre code va bien évidemment devoir utiliser un peu d’assembleur. Et bien que ce soit relativement aisé, grâce à un système d’assembleur *inline* très proche de celui utilisé en C, c’est également tout sauf sûr, alors vous allez devoir prévenir le compilateur que vous comptez l’utiliser, pour qu’il puisse se préparer psychologiquement.

.. code:: rust

    #![no_std]

Toujours évidemment, notre code ne doit *surtout* pas être lié à la bibliothèque standard de Rust, qui elle-même utilise la libc. On peut malgré tout utiliser la bibliothèque ``core``, qui contient tout un tas de choses utiles et dénuées de dépendance au système (sauf pour l’allocation de mémoire), comme des conteneurs ou des chaînes de caractères intelligentes.

.. code:: rust

    pub enum Syscall    {
        Useless(u32, Option<u32>)
    }

Maintenant on attaque le vif du sujet, en définissant un type pour décrire nos appels système. Mais ne vous laissez pas avoir par le mot-clé : une ``enum`` en Rust n’a pas grand chose à voir avec une ``enum`` en C, ça se rapproche bien plus d’un type algébrique de données de Haskell et consorts.

En d’autres termes, une ``enum`` est un type qui peut avoir plusieurs formes, chaque forme pouvant contenir à peu près n’importe quoi, du moment que ce n’importe quoi est toujours du même type. Ici, notre ``enum`` n’a qu’une variante, mais on pourrait imaginer de porter les deux appels système ``write()`` et ``exit()`` (les seuls dont on ait besoin pour un Hello world basique), et voilà ce que cela donnerait.

.. code:: rust

    pub enum Syscall    {
        Exit(i32),
        Write(i32, *const u8, usize)
    }

En réalité, ce serait une transposition assez nulle et, par exemple, la seconde variante donnerait plutôt quelque chose comme ``Write(&File, String)``, le type ``File`` restant à définir par ailleurs. Mais retournons à notre code.

.. code:: rust

    pub enum Syscall    {
        Useless(u32, Option<u32>)
    }

Vous l’avez sans doute deviné, ``u32`` est un entier non signé sur 32 bits, c’est-à-dire un ``unsigned int`` de C. Et vous avez vu tantôt qu’il existe aussi des ``u8`` (entier non signé sur 8 bits), des ``i32`` (entier signé sur 32 bits), et toute la famille jusqu’à ``u64``/``i64``. Et pour aller jusqu’au bout, ``usize`` et ``isize`` sont des entiers dont la largeur permet de contenir un pointeur sur l’architecture en question, soit 64 bits sur x86_64.

De son côté, ``Option<u32>`` vous est sans doute tout sauf familier si vous ne pratiquez usuellement que le C : c’est l’équivalent exact du type ``Maybe`` de Haskell, c’est-à-dire un type pour représenter une valeur qui peut être définie ou non. Essayons d’être plus clair.

``Option<T>`` est (1) un type paramétré et (2) une ``enum`` à deux variantes, ``None`` et ``Some(T)``. Par (1) on entend que ``Option`` est un type générique, qui doit être spécialisé en spécifiant le type qu’il va contenir, grâce à la syntaxe ``<>`` (on trouve la même en C++).

En clair, ``Option<u32>`` signifie « Un ``u32`` ou rien du tout, ça dépend. ». Ici, on va l’utiliser pour la valeur de retour de l’appel système : tant que l’appel système n’a pas été appelé, il vaut ``None``, puis il est remplacé par ``Some(resultat_de_l_appel_systeme)``.

Ce type est très puissant, et permet de se débarraser des pointeurs nuls, des valeurs bateaux pour qu’un argument ne soit pas pris en compte, et des structures ayant des champs optionnels, entre autres.

Pour finir, the mot-clé ``pub`` rend le type public, ce qui revient plus ou moins à exporter le symbole. Le fonctionnement est plus compliqué que ça, mais pour cet exemple, vous n’avez pas besoin d’en comprendre plus.

La suite risque d’être plus difficile à digérer. Alors allons-y petit bout par petit bout.

.. code:: rust

    impl Syscall    {
        #[cfg(feature = "userspace")]
        pub fn call(&mut self)  {
            let pointer = self as *mut Syscall;

            if (pointer as usize) < 1024    {
                // Ça ne devrait pas arriver, mais il faudrait
                // prévoir quelque chose au cas où.
            }

            unsafe { asm_syscall(pointer); }
        }

        #[cfg(feature = "kernel")]
        pub fn handle(&mut self)    {
            match *self {
                Syscall::Useless(data, ref mut ret) => *ret = Some(data + 15),
            };
        }
    }

Le bloc ``impl Syscall { }`` sert à implémenter des méthodes pour notre type ``Syscall``. Là encore, ne vous laissez pas abuser par le nom, c’est n’est pas un système complet de POO à la C++, avec de l’héritage et tout. En Rust, les méthodes sont juste des fonctions intégrées à un espace de noms, qui peuvent être appelées avec la syntaxe de méthode ``var.methode(args)`` plutôt que la syntaxe classique ``Type::methode(var, args)``. Elles permettent également d’utiliser ``self`` comme premier argument, qui doit être du type pour lequel vous implémentez.

Donc, ``pub fn call(&mut self)`` est une fonction publique qui prend comme argument une référence mutable (cf. infra) vers un ``Syscall``, et peut être appelée au moyen de ``mon_appel.call()``, qui est quand même plus lisible que ``Syscall::call(&mut mon_appel)``.

Quid de la référence mutable, alors ? Ben, tout d’abord, une référence est plus ou moins un pointeur sécurisé : on ne peut pas créer une référence vers une variable inexistante, une référence ne peut pas se retrouver à ne pointer vers rien, et il en existe deux sortes, et c’est là qu’on parle de mutabilité.

En Rust, les variables sont non mutables par défaut, on ne peut pas changer leur valeur après leur en avoir assigné une. Si vous voulez qu’elles soient mutables, il faut le déclarer explicitement au moyen du mot-clé ``mut``. Il en va de même des références : elles peuvent être non mutables, alias en lecture seule (``&variable``), ou mutables, alias en lecture-écriture (``&mut variable``), à condition bien sûr de pointer vers une variable mutable.

Cela peut paraître superfétatoire, mais il y a une subtilité : il ne peut y avoir qu’une seule référence mutable à la fois à une même variable, et on ne peut avoir en même temps une référence mutable et des références non mutables vers une variable donnée. Ainsi, tout accès concurrent aux données est impossible.

Mais intéressons-nous à ces belles lignes de code.

.. code:: rust

    #[cfg(feature = "userspace")]

    #[cfg(feature = "kernel")]

C’est la compilation conditionnelle à la sauce Rust. Rust permet de triturer le code dans tous les sens en fonction de tout un tas de paramètres, et il est hors de question que je détaille le sujet : jetez juste un œil au `manuel officiel d’apprentissage`__ et à la `référence du langage`__ (liens en anglais).

.. __: https://doc.rust-lang.org/book/conditional-compilation.html
.. __: https://doc.rust-lang.org/reference/attributes.html#conditional-compilation

Ce qu’on a là est l’équivalent des ``#ifdef CONFIG_NAWAK`` en pagaille que l’on trouve dans le code du noyau. Il suffit d’appeler le compilateur Rust avec le paramètre ``--cfg feature=\"kernel\"`` pour que tous les blocs marqués de ``#[cfg(feature = "kernel")]`` soient inclus dans le code, tandis que ceux qui n’auront pas reçu leur ``--cfg feature=truc`` disparaîtront simplement du code.

En quoi cela peut-il nous être utile dans le contexte actuel ? Eh bien, cela signifie que l’on n’a qu’un seul code source pour à la fois le noyau et la bibliothèque standard qui utilisera le noyau, la seule différence étant une option de compilation. De cette manière, vous avez la garantie que les deux restent cohérents entre eux.

À présent, suivons le cours de notre appel système, depuis l’appel dans l’espace utilisateur jusqu’à son traitement dans l’espace noyau.

.. code:: rust

        pub fn call(&mut self)  {
            let pointer = self as *mut Syscall;

            if (pointer as usize) < 1024    {
                // Ça ne devrait pas arriver, mais il faudrait
                // prévoir quelque chose au cas où.
            }

            unsafe { asm_syscall(pointer); }
        }

La fonction ``call()`` prend donc pour seul argument une référence mutable vers un ``Syscall``. Et elle commence par en faire un pointeur mutable à la C (``*mut Syscall``, ``let`` étant le mot-clé pour définir une variable). Cette sorte de pointeurs fait bien sûr perdre tous les bienfaits des références, et elle est par conséquent considéré comme dangereuse dans la plupart de ses usages, mais a l’avantage de ne contenir d’autre information que l’adresse vers laquelle elle pointe.

Le bloc suivant est un bloc conditionnel, dont vous devriez comprendre aisément la syntaxe. Quant à son utilité, on y reviendra prochainement.

Enfin, la fonction ``asm_syscall()`` est appelée sur le pointeur nu, et ce qui s’y passe est totalement *unsafe*. Alors il faut mettre l’appel dans un bloc ``unsafe { }``, pour indiquer au compilateur que, oui, c’est dangereux, mais on prend la responsabilité de tout ce qui peut advenir, parce qu’on sait ce qu’on fait.

À présent, direction cette fonction ``asm_syscall()``.

.. code:: rust

    #[cfg(all(feature = "userspace", target_arch = "x86_64"))]
    #[inline(always)]
    pub unsafe fn asm_syscall(ptr : *mut Syscall)   {
        asm!("syscall"
         : : "{rax}"(ptr)
           : "rcx", "r11", "memory"
           : "intel", "volatile"
        );
    }

Premièrement, vous avez remarqué le bloc de configuration ? Le ``all()`` est un ET logique, les deux conditions doivent être remplies : on doit avoir demandé la fonctionnalité ``userspace``, et l’architecture cible doit être x86_64. Ce qui tient naturellement au fait que nous mettons de l’assembleur x86_64 dans la fonction.

Deuxièmement, vous n’aurez sans doute aucun mal à comprendre ``#[inline(always)]``. Cette fonction est une unique instruction d’assembleur, cela n’aurait pas de sens de perdre son temps à faire l’aller-retour jusqu’à elle. Mais cette unique instruction dépend de l’architecture pour laquelle on compile, alors c’est plus simple d’en faire une fonction à part et soumise à compilation conditionnelle que d’écrire une variante de ``call()`` pour chaque achitecture.

Troisièmement, voyez le mot-clé ``unsafe`` dans la définition de la fonction : cela indique que tout le corps de la fonction est dangereux. On pourrait aussi l’écrire ainsi. Mais ce serait stupide.

.. code:: rust

    #[cfg(all(feature = "userspace", target_arch = "x86_64"))]
    #[inline(always)]
    pub fn asm_syscall(ptr : *mut Syscall)  {
        unsafe  {
            asm!("syscall"
             : : "{rax}"(ptr)
               : "rcx", "r11", "memory"
               : "intel", "volatile"
            );
        }
    }

Pour finir, le bloc ``asm!()`` a plus ou moins la même syntaxe que l’assembleur *inline* de C, donc je ne détaillerai pas beaucoup. Basiquement, ça dit que l’instruction ``syscall`` doit être exécutée ; qu’avant cela, ``ptr`` doit être mis dans le registre ``rax`` ; que les registres ``rcx`` et ``r11`` ainsi que la RAM vont être affectés par le code assembleur, donc le compilateur de Rust ne doit rien présumer de leur valeur après que le bloc aura été exécuté ; que j’utilise la syntaxe Intel plutôt que la syntaxe AT&T dont gas est friand (laquelle est objectivement de la bouse) ; que le code doit rester à cette place précise et ne surtout pas être déplacé à des fins d’optimisation.

Et finalement, l’appel système est lancé, le noyau va prendre la main ! Mais que s’est-il passé au juste ? Au lieu de mettre un numéro d’appel système dans ``rax``, puis ses arguments dans d’autres registres, et de finalement récupérer le retour dans ``rax`` à nouveau, on a mis l’adresse du ``Syscall`` dans ``rax`` puis déclenché l’appel système avec cette seule information.

Cette adresse peut être comprise entre ``0`` et ``0xffffffffffffffff``, alors que la numérotation des appels système s’arrête un peu après 512. Ce qui signifie que, à moins d’avoir la grande malchance que le ``Syscall`` ait une adresse inférieure à 1024 (en fait, ça ne devrait jamais arriver, cette zone de la mémoire est censée être utilisée par le noyau), il ne pourra jamais être confondu avec un appel système à l’ancienne en C.

Le but de tout cela, c’est qu’un petit bout de magie assembleur (que l’on verra plus tard) va séparer le bon grain de nos appel système avec ``rax`` supérieur à 1024, qui seront envoyés vers notre gestionnaire écrit en Rust, de l’ivraie des appels système à l’ancienne avec ``rax`` inférieur à 1024, qui seront délégués au gestionnaire préexistant. Et c’est là notre prochaine étape.

.. code:: rust

    #[cfg(all(feature = "kernel", target_arch = "x86_64"))]
    #[no_mangle]
    pub unsafe extern fn rust_syscall_handle() {
        let pointer : *mut Syscall;

        asm!(""
           : "={rax}"(pointer)
         : : "memory"
           : "intel", "volatile"
        );

        (*pointer).handle();
    }

Je n’insiste pas sur la ligne de configuration, vous l’avez déjà comprise. Passons à ``#[no_mangle]``. Rust est comme C++, il trafique le nom des symboles qu’il exporte, si bien que notre méthode ``handle()`` de tantôt sera en réalité exportée comme un truc dans les tons de ``_ZN7syscall7Syscall6handle17hd86ed5a2914e5b27E``, et ce nom change chaque fois que le code est compilé. Alors comment peut-on bien appeler une telle fonction depuis du code C ? En disant au compilateur de Rust de ne pas toucher au nom. Au moyen de ``#[no_mangle]``.

La ligne de définition de la fonction contient un nouveau mot-clé : ``extern``. Rust utilise une convention d’appel maison pour ses fonctions, qui n’est pas celle utilisée par C. Alors si vous voulez qu’une fonction donnée utilise une autre convention d’appel que celle standard de Rust, il faut le préciser au moyen de ``extern "nom_de_la_convention"``. Mais pour simplifier les choses, et parce que c’est le cas le plus courant, ``extern`` tout seul est synonyme de ``extern "C"``.

Ce qui arrive ensuite, c’est que le contenu du registre ``rax`` est remis dans la variable ``pointer``. Puis le pointeur nu est déréférencé (``*pointer``), ce qui est totalement *unsafe*, sachez-le, et pour finir, la méthode ``handle()`` est appelée sur le ``Syscall`` qui en résulte.

.. code:: rust

    #[cfg(feature = "kernel")]
    pub fn handle(&mut self)    {
        match *self {
            Syscall::Useless(data, ref mut ret) => *ret = Some(data + 15),
        };
    }

De nouveau, la fonction prend une référence mutable vers un ``Syscall`` en argument. Si vous suivez correctement, c’est exactement le même ``Syscall`` que celui qu’on a utilisé avec la méthode ``call()`` : même emplacement en mémoire, même contenu, et tout. Dit autrement, on a envoyé tous les arguments de l’appel système au noyau sans jamais les mettre dans des registres.

Ensuite, on filtre par motifs le ``Syscall`` qu’on vient de recevoir. Cela aussi vient de la programmation fonctionnelle et n’est autre que le revers de la pièce appelée « types algébriques de données », et c’est ce qui le rend si foutrement puissant. Ça ressemble à un ``switch`` des langages comme C, mais dopé à l’huile essentielle de coke.

Pour expliquer cela simplement (du moins essayer), on va tester la variable contre chaque variante de l’\ ``enum``, puis exécuter le bloc ou l’instruction qui suit le ``=>`` correspondant. Juste pour votre information, Rust vous oblige à définir un comportement pour chaque variante d’une ``enum``, pour s’assurer qu’il n’y ait pas de comportement indéfini. Et que, lorsque vous ajoutez une nouvelle variante, un comportement lui a bien été défini à chaque endroit où vous utilisez l’\ ``enum``.

Ceci étant dit, c’est bien plus que cela : vous pouvez d’un coup d’un seul associer le contenu d’une variante à des variables locales, que vous pourrez utiliser à leur tour dans le bras droit de ``=>``. Ici, notre ``u32`` est mis dans ``data``, et on définit une référence mutable vers notre ``Option<u32>``, que l’on appelle ``ret``.

Parce que cette ``Option<u32>`` va être utilisée comme valeur de retour de l’appel système, comme je l’ai expliqué beaucoup plus haut. Enfin, ``*ret = Some(data + 15)`` se contente de faire en sorte que le ``None`` contenu dans le ``Syscall`` d’origine soit remplacé par un ``Some()`` contenant l’argument de l’appel + 15. Ben ouais, il s’appelle pas ``Useless`` pour rien…

Si bien que, lorsque le cours du code sera renvoyé dans l’espace utilisateur par un peu plus de magie assembleur, le dernier contenu du ``Syscall`` ne dira plus « Circulez, y’a rien à voir. » mais « Regarde ! J’ai ta réponse ! ».

Makefile
========

Tout ce qu’il vous manque pour pleinement comprendre comment les appels système nouveau style fonctionnent, c’est la partie en assembleur. Mais d’abord, nous allons voir comment intégrer le code en Rust dans un noyau écrit tout en C. Première chose à savoir, le code qui fait l’interface entre l’espace utilisateur et le noyau sur une machine x86 ou x86_64 se trouve dans le dossier ``/arch/x86/entry`` (code source complet pour le noyau utilisé ici sur `Elixir`__).

.. __: http://elixir.free-electrons.com/linux/v4.8.17/source/arch/x86/entry

Il nous faut donc modifier le Makefile de ce répertoire précis, dont je vous offre le contenu complet.

.. code:: make

    #
    # Makefile for the x86 low level entry code
    #

    OBJECT_FILES_NON_STANDARD_entry_$(BITS).o   := y
    OBJECT_FILES_NON_STANDARD_entry_64_compat.o := y

    CFLAGS_syscall_64.o		+= $(call cc-option,-Wno-override-init,)
    CFLAGS_syscall_32.o		+= $(call cc-option,-Wno-override-init,)
    obj-y				:= entry_$(BITS).o thunk_$(BITS).o syscall_$(BITS).o
    obj-y				+= common.o

    obj-y				+= vdso/
    obj-y				+= vsyscall/

    obj-$(CONFIG_IA32_EMULATION)	+= entry_64_compat.o syscall_32.o

Cela n’a rien d’intuitif, alors avant de poursuivre votre lecture, allez lire cette `superbe explication`__ (en anglais). Oui, elle date de 2003 et traite du noyau 2.5, mais étonnamment, elle est toujours valide.

.. __: https://lwn.net/Articles/21835/

Et voici le Makefile modifié que j’ai utilisé.

.. code:: make

    #
    # Makefile for the x86 low level entry code
    #

    OBJECT_FILES_NON_STANDARD_entry_$(BITS).o   := y
    OBJECT_FILES_NON_STANDARD_entry_64_compat.o := y

    $(obj)/rs-syscall.o: $(src)/syscall.rs
	    rustc -O --cfg feature=\"kernel\" -C prefer-dynamic $(src)/syscall.rs
	    ar x libsyscall.rlib syscall.0.o
	    mv syscall.0.o $(src)/rs-syscall.o
	    rm libsyscall.rlib
	    rustc -O --cfg feature=\"userspace\" -C prefer-dynamic $(src)/syscall.rs

    CFLAGS_syscall_64.o		+= $(call cc-option,-Wno-override-init,)
    CFLAGS_syscall_32.o		+= $(call cc-option,-Wno-override-init,)
    obj-y				:= entry_$(BITS).o thunk_$(BITS).o syscall_$(BITS).o
    obj-y				+= common.o
    obj-y				+= rs-syscall.o

    obj-y				+= vdso/
    obj-y				+= vsyscall/

    obj-$(CONFIG_IA32_EMULATION)	+= entry_64_compat.o syscall_32.o

Premièrement, l’objet ``rs-sycall.o`` a été ajouté à la liste des fichiers objet nécessaires pour compiler le noyau. Ensuite, on dit que ce fichier objet dépend du code source ``syscall.rs``, et on décrit la procédure pour en tirer un fichier objet.

Le compilateur de Rust peut produire plusieurs types de résultat : un binaire pleinement exécutable, une bibliothèque dynamique (``libxxxx.so``), une bibliothèque statique C classique (``libxxxx.a``), ou une bibliothèque Rust alias rlib (``libxxxx.rlib``). Les deux premières lignes du code source, que nous n’avons pas encore abordées, servent à dire au compilateur qu’on souhaite obtenir une rlib, et l’appeler ``libsyscall.rlib``.

.. code:: rust

    #![crate_type = "rlib"]
    #![crate_name = "syscall"]

Une rlib est un format de sortie très intéressant, parce qu’il contient le code compilé, mais également tous les types, fonctions, etc. exportés. En clair, elle sert à la fois de bibliothèque statique et de fichier d’en-tête.

En outre, ce n’est rien de bien compliqué : en réalité, c’est juste une archive ar contenant des fichiers objets, exactement comme une bibliothèque statique C classique. Et il se trouve que le fichier objet contenant notre code complet s’appelle ``syscall.0.o``.

Du coup, ``rustc -O --cfg feature=\"kernel\" -C prefer-dynamic $(src)/syscall.rs`` compile le code, restreint aux bouts qui constituent le noyau, avec des optimisations (``-O``), et en le liant dynamiquement à la bibliothèque standard de Rust, ou plus exactement à la bibliothèque ``core`` (``-C prefer-dynamic``), de sorte qu’on n’ait que les fichiers objet correspondant à notre code dans ``libsyscall.rlib``.

Ensuite, on extrait ``syscall.0.o`` de la rlib, on le déplace dans ``/arch/x86/entry`` tout en le renommant ``rs-syscall.o``, avant de supprimer la rlib.

Et pour finir, on recompile la rlib, mais cette fois uniquement les bouts qui appartiennent à l’espace utilisateur, ce qui nous fait une jolie petite « bibliothèque standard » pour plus tard. Que l’on trouvera à la racine du code source.

entry_64.S
==========

À présent, respirez un bon coup, parce qu’on va mettre les mains dans du code assembleur. Et du code assembleur écrit en sytaxe AT&T qui, comme précédemment signalé, est une sacrée bouse. Le fichier d’origine fait presque 1500 lignes de long, alors je ne mettrai ici que la partie qui va nous intéresser.

.. code:: gas

    /*
     * 64-bit SYSCALL instruction entry. Up to 6 arguments in registers.
     *
     * […]
     */

    ENTRY(entry_SYSCALL_64)
	    /*
	     * Interrupts are off on entry.
	     * We do not frame this tiny irq-off block with TRACE_IRQS_OFF/ON,
	     * it is too small to ever cause noticeable irq latency.
	     */
	    SWAPGS_UNSAFE_STACK
	    /*
	     * A hypervisor implementation might want to use a label
	     * after the swapgs, so that it can do the swapgs
	     * for the guest and jump here on syscall.
	     */
    GLOBAL(entry_SYSCALL_64_after_swapgs)

	    movq	%rsp, PER_CPU_VAR(rsp_scratch)
	    movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp

	    TRACE_IRQS_OFF

	    /* Construct struct pt_regs on stack */
	    pushq	$__USER_DS			/* pt_regs->ss */
	    pushq	PER_CPU_VAR(rsp_scratch)	/* pt_regs->sp */
	    pushq	%r11				/* pt_regs->flags */
	    pushq	$__USER_CS			/* pt_regs->cs */
	    pushq	%rcx				/* pt_regs->ip */
	    pushq	%rax				/* pt_regs->orig_ax */
	    pushq	%rdi				/* pt_regs->di */
	    pushq	%rsi				/* pt_regs->si */
	    pushq	%rdx				/* pt_regs->dx */
	    pushq	%rcx				/* pt_regs->cx */
	    pushq	$-ENOSYS			/* pt_regs->ax */
	    pushq	%r8				/* pt_regs->r8 */
	    pushq	%r9				/* pt_regs->r9 */
	    pushq	%r10				/* pt_regs->r10 */
	    pushq	%r11				/* pt_regs->r11 */
	    sub	$(6*8), %rsp			/* pt_regs->bp, bx, r12-15 not saved */

	    /*
	     * If we need to do entry work or if we guess we'll need to do
	     * exit work, go straight to the slow path.
	     */
	    testl	$_TIF_WORK_SYSCALL_ENTRY|_TIF_ALLWORK_MASK, ASM_THREAD_INFO(TI_flags, %rsp, SIZEOF_PTREGS)
	    jnz	entry_SYSCALL64_slow_path

    entry_SYSCALL_64_fastpath:
	    /*
	     * Easy case: enable interrupts and issue the syscall.  If the syscall
	     * needs pt_regs, we'll call a stub that disables interrupts again
	     * and jumps to the slow path.
	     */
	    TRACE_IRQS_ON
	    ENABLE_INTERRUPTS(CLBR_NONE)
    #if __SYSCALL_MASK == ~0
	    cmpq	$__NR_syscall_max, %rax
    #else
	    andl	$__SYSCALL_MASK, %eax
	    cmpl	$__NR_syscall_max, %eax
    #endif
	    ja	1f				/* return -ENOSYS (already in pt_regs->ax) */
	    movq	%r10, %rcx

	    /*
	     * This call instruction is handled specially in stub_ptregs_64.
	     * It might end up jumping to the slow path.  If it jumps, RAX
	     * and all argument registers are clobbered.
	     */
	    call	*sys_call_table(, %rax, 8)
    .Lentry_SYSCALL_64_after_fastpath_call:

	    movq	%rax, RAX(%rsp)
    1:

	    /*
	     * If we get here, then we know that pt_regs is clean for SYSRET64.
	     * If we see that no exit work is required (which we are required
	     * to check with IRQs off), then we can go straight to SYSRET64.
	     */
	    DISABLE_INTERRUPTS(CLBR_NONE)
	    TRACE_IRQS_OFF
	    testl	$_TIF_ALLWORK_MASK, ASM_THREAD_INFO(TI_flags, %rsp, SIZEOF_PTREGS)
	    jnz	1f

	    LOCKDEP_SYS_EXIT
	    TRACE_IRQS_ON		/* user mode is traced as IRQs on */
	    movq	RIP(%rsp), %rcx
	    movq	EFLAGS(%rsp), %r11
	    RESTORE_C_REGS_EXCEPT_RCX_R11
	    movq	RSP(%rsp), %rsp
	    USERGS_SYSRET64

    1:
	    /*
	     * The fast path looked good when we started, but something changed
	     * along the way and we need to switch to the slow path.  Calling
	     * raise(3) will trigger this, for example.  IRQs are off.
	     */
	    TRACE_IRQS_ON
	    ENABLE_INTERRUPTS(CLBR_NONE)
	    SAVE_EXTRA_REGS
	    movq	%rsp, %rdi
	    call	syscall_return_slowpath	/* returns with IRQs disabled */
	    jmp	return_from_SYSCALL_64

Oui, c’est immonde, et en tout honnêteté, je ne comprends pas totalement ce qui se passe. C’est sans doute en partie dû à la syntaxe AT&T qui est… bon, je pense que vous avez compris l’idée. Juste après qu’on a exécuté l’instruction ``syscall`` dans l’espace utilisateur et que le processeur a effectué sa magie vaudou pour rendre la main au noyau, le cours du code reprend à cet endroit précis : ``ENTRY(entry_SYSCALL_64)``.

Tout le long jusqu’à ``sub $(6*8), %rsp``, le code sauve plus ou moins l’état de la machine pour être en mesure de le rétablir à la fin de l’appel système. Puis il réalise un test pour potentiellement passer à une procédure d’entrée différente, et c’est là que j’ai introduit la première partie de mon code.

.. code:: gas

    /* Le gestionnaire d’appels système de Rust */
    cmpq	$1024, %rax
    jae	rust_entry_syscall

C’est en vérité très simple : on compare la valeur de ``rax`` à 1024, et si c’est supérieur ou égal à 1024, on saute à l’étiquette ``rust_entry_syscall``. En d’autres termes, comme décrit tantôt, si l’appel système a été appelé depuis Rust et que ``rax`` contient un pointeur vers un ``Syscall``, on bifurque vers notre gestionnaire d’appels système en Rust, sinon, on laisse le code se dérouler comme prévu.

La suite du code ne s’arrête jamais jusqu’à ``jmp return_from_SYSCALL_64`` : ceci est un saut inconditionnel, ce qui signifie qu’on peut mettre du code juste après sans aucun risque que le cours du code s’y retrouve sans qu’on y ait explicitement sauté. Et c’est ainsi que j’introduis la deuxième partie du gestionnaire d’appels système en Rust.

.. code:: gas

    rust_entry_syscall:
	    TRACE_IRQS_ON
	    ENABLE_INTERRUPTS(CLBR_NONE)

	    call	rust_syscall_handle

	    jmp	.Lentry_SYSCALL_64_after_fastpath_call

Pour bien comprendre, voici une version résumée de ce qui se passe quand un appel système à l’ancienne mode est appelé.

.. code:: gas

    /* */
	    TRACE_IRQS_ON
	    ENABLE_INTERRUPTS(CLBR_NONE)
	    call	*sys_call_table(, %rax, 8)
    .Lentry_SYSCALL_64_after_fastpath_call:

Je ne comprends pas exactement ce que font les deux premières lignes, mais il est évident qu’elles sont nécessaires, alors je les reproduis dans mon propre code. Puis, la fonction gestionnaire associée au numéro d’appel système est appelée. Dans ma version, au contraire, j’appelle la ``pub unsafe extern fn rust_syscall_handle()`` dont nous avons déjà parlé.

Et finalement, quand la fonction est arrivée à son terme, je saute pour retourner à l’étiquette qui suit immédiatement l’appel au gestionnaire d’appel système dans le code d’origine, et je laisse ledit code d’origine gérer le retour dans l’espace utilisateur. J’ai essayé d’écrire mon propre code plus concis, mais je me suis lamentablement foiré, alors contentons-nous de la solution de facilité pour le moment.

À présent, vous avez la chaîne complète depuis Rust qui réalise un appel système, jusqu’à Rust qui traite ce même appel système dans le noyau, puis retour à l’espace utilisateur. Dooooonc… il est temps de voir cela en action !

Ça marche !
===========

.. code:: rust

    extern crate syscall;

    use syscall::Syscall;

    fn main()   {
        let mut lets_be_useless = Syscall::Useless(15, None);
        lets_be_useless.call();

        let Syscall::Useless(_, ref result) = lets_be_useless;
        println!("L’appel système a renvoyé : {}", result.unwrap());
    }

Vous devriez à présent être capables de comprendre la plupart de ce que fait ce code. ``extern crate syscall;`` est plus ou moins équivalent à ``#include "syscall.h"``, sauf que les fonctions, types, etc. sont dans un espace de noms, alors ``use syscall::Syscall;`` nous débarrasse des ``syscall::`` pour le moins pénibles qu’il nous faudrait placer devant chaque occurrence de ``Syscall``.

Si vous vous rappelez bien, la valeur de retour de l’appel système est un ``Option<u32>`` : on définit une référence non mutable qui y pointe et qu’on appelle ``result``. Puis, ``result.unwrap()`` est une fonction standard qui renvoie le contenu du ``Some()`` et panique si la valeur est ``None``. Mais ici, on est certains que c’est un ``Some()``.

Et voilà le résultat.

.. code:: console

    carnufex@KAMARAD-PC $ ./test
    L’appel système a renvoyé : 30

Alors évidemment, le code est loin d’être parfait. Par exemple, au lieu d’une ``Option``, l’appel système devrait renvoyer un ``Result``, qui permet de renvoyer soit (vous ai-je dit que ce type s’appelle ``Either`` en Haskell ? :3) un résultat effectif, soit un code d’erreur, et de savoir duquel il s’agit. En lieu et place du système à chier utilisé en C, alias « Une valeur négative et basse est l’inverse du code d’erreur, tandis qu’une valeur positive ou bien  négative mais haute est un vrai résultat. ».

Et le morceau en assembleur est plutôt crade, il se peut que je n’aie pas pensé à tout. Et pour être parfaitement honnête, j’ignore totalement si cette façon de procéder introduit des failles de sécurité, même si j’en doute, étant donné que les appels système en C utilisent des pointeurs vers l’espace utilisateur tout le temps. Et j’ignore aussi si mon code est *thread-safe* ou non.

Cette méthode présente également quelques inconvénients. Le plus important, à mon sens, est que Rust ne donne absolument aucune garantie quant à la représentation interne de ses ``enum``. Cela veut dire que, lorsque vous ajoutez de nouvelles variantes à l’\ ``enum``, la valeur numérique effectivement associée à chaque variante est susceptible de changer.

De ce fait, du code de l’espace utilisateur compilé avec une version du code source pourrait ne pas être compatible avec un noyau compilé avec une autre version du code. Le fait que la bibliothèque standard soit compilée en même temps que le noyau et à partir du même code source atténue le problème, tant que les logiciels sont liés dynamiquement à la bibliothèque standard.

D’un autre côté, les logiciels liés statiquement risquent d’avoir besoin qu’on les recompile pour continuer de fonctionner. Une manière de contourner le problème serait de ne jamais changer l’ordre des variantes, de seulement en ajouter de nouvelles à la fin de l’\ ``enum``. De cette sorte, les numéros associés aux anciennes variantes devraient rester les mêmes d’une version à l’autre.

Par ailleurs, le code du noyau devra se montrer très prudent s’il désire renvoyer un type conteneur comme ``String`` ou ``Vec``. En effet, un ``String`` n’est en réalité qu’un tableau de ``char`` alloué sur le tas d’un côté, un pointeur vers ce tableau et quelques infos supplémentaires comme sa longueur ou sa capacité d’accueil alloués sur la pile de l’autre.

Et si un ``String`` est renvoyé comme valeur de retour de l’appel système, seule la partie pile sera effectivement écrite dans la mémoire de l’espace utilisateur, et le tableau de ``char`` restera là où il a été alloué par le noyau, c’est-à-dire dans la mémoire de l’espace noyau. Et lorsque du code de l’espace utilisateur tentera d’y accéder, il va se prendre une erreur de segmentation dans la tête. Il est possible d’éviter ce problème, mais cela requiert une attention particulière de la part des développeurs.

Conclusion
==========

Mais tout cela n’est que détails. L’important, c’est que ça marche effectivement pour de vrai sur un noyau de la vraie vie, et cela ouvre la voie à la possibilité de progressivement réécrire des bouts du noyau en Rust, offrant ainsi un code plus sûr et plus expressif. Ô joie !

Mon œuvre est à présent achevée. N’hésitez pas à commenter, m’insulter moi ou ma mère, ou m’envoyer des photos de nichons, selon ce qui vous paraîtra le plus approprié.

-----

Détail légal : j’ignore totalement d’où vient le Tux tout mignon du logo. S’il est à vous et que vous ne voulez pas le voir ici, n’hésitez pas à m’en parler.
