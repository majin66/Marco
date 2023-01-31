Le DOM virtuel est une pure surcharge :

Retirons une fois pour toutes le mythe du « DOM virtuel est rapide »

RICHE HARRIS 27 DÉCEMBRE 2018

Si vous avez utilisé des frameworks JavaScript au cours des dernières années, vous avez probablement entendu l'expression "le DOM virtuel est rapide", souvent censée signifier qu'il est plus rapide que le vrai DOM. C'est un mème étonnamment résistant - par exemple, les gens ont demandé comment Svelte peut être rapide lorsqu'il n'utilise pas de DOM virtuel.

Il est temps de regarder de plus près.

Qu'est-ce que le DOM virtuel ?
Dans de nombreux frameworks, vous construisez une application en créant render()des fonctions, comme ce simple composant React :

function HelloMessage(props) {
    return (
        <div className="greeting">
            Hello {props.name}
        </div>
    );
}
Vous pouvez faire la même chose sans JSX...

function HelloMessage(props) {
    return React.createElement(
        'div',
        { className: 'greeting' },
        'Hello ',
        props.name
    );
}
... mais le résultat est le même — un objet représentant à quoi la page devrait maintenant ressembler. Cet objet est le DOM virtuel. Chaque fois que l'état de votre application est mis à jour (par exemple lorsque l' nameaccessoire change), vous en créez un nouveau. Le travail du framework est de réconcilier le nouveau avec l'ancien, de déterminer quels changements sont nécessaires et de les appliquer au vrai DOM.

Comment le mème a-t-il commencé ?
Les affirmations mal comprises concernant les performances du DOM virtuel remontent au lancement de React. Dans Rethinking Best Practices , une conférence phare de 2013 par Pete Hunt, ancien membre de l'équipe principale de React, nous avons appris ce qui suit :


C'est en fait extrêmement rapide, principalement parce que la plupart des opérations DOM ont tendance à être lentes. Il y a eu beaucoup de travail de performance sur le DOM, mais la plupart des opérations DOM ont tendance à perdre des images.


Mais attendez une minute ! Les opérations du DOM virtuel s'ajoutent aux éventuelles opérations sur le DOM réel. La seule façon d'être plus rapide est de le comparer à un cadre moins efficace (il y en avait beaucoup à faire en 2013 !), Ou de se disputer contre un homme de paille - que l'alternative est de faire quelque chose que personne ne fait réellement :

onEveryStateChange(() => {
    document.body.innerHTML = renderMyApp();
});
Pete clarifie peu après...

Réagir n'est pas magique. Tout comme vous pouvez passer à l'assembleur avec C et battre le compilateur C, vous pouvez passer aux opérations DOM brutes et aux appels d'API DOM et battre React si vous le souhaitez. Cependant, l'utilisation de C ou Java ou JavaScript est une amélioration des performances d'un ordre de grandeur car vous n'avez pas à vous soucier... des spécificités de la plate-forme. Avec React, vous pouvez créer des applications sans même penser aux performances et l'état par défaut est rapide.

... mais ce n'est pas la partie qui coince.

Alors... le DOM virtuel est- il lent ?
Pas exactement. Cela ressemble plus à "le DOM virtuel est généralement assez rapide", mais avec certaines mises en garde.

La promesse initiale de React était que vous pouviez restituer l'intégralité de votre application à chaque changement d'état sans vous soucier des performances. En pratique, je ne pense pas que cela se soit avéré exact. Si c'était le cas, il n'y aurait pas besoin d'optimisations comme shouldComponentUpdate(ce qui est une façon de dire à React quand il peut ignorer un composant en toute sécurité).

Même avec shouldComponentUpdate, la mise à jour du DOM virtuel de toute votre application en une seule fois représente beaucoup de travail. Il y a quelque temps, l'équipe React a introduit quelque chose appelé React Fiber qui permet de diviser la mise à jour en plus petits morceaux. Cela signifie (entre autres) que les mises à jour ne bloquent pas le thread principal pendant de longues périodes, bien que cela ne réduise pas la quantité totale de travail ou la durée d'une mise à jour.

D'où vient le surcoût ?
De toute évidence, différer n'est pas gratuit . Vous ne pouvez pas appliquer de modifications au DOM réel sans d'abord comparer le nouveau DOM virtuel avec l'instantané précédent. Pour reprendre l'exemple précédent HelloMessage, supposons que l' nameaccessoire passe de « monde » à « tout le monde ».

Les deux instantanés contiennent un seul élément. Dans les deux cas, c'est un <div>, ce qui signifie que nous pouvons garder le même nœud DOM
Nous énumérons tous les attributs de l'ancien <div>et du nouveau pour voir si certains doivent être modifiés, ajoutés ou supprimés. Dans les deux cas, nous avons un seul attribut — a classNameavec une valeur de"greeting"
En descendant dans l'élément, nous voyons que le texte a changé, nous devrons donc mettre à jour le vrai DOM
De ces trois étapes, seule la troisième a de la valeur dans ce cas, puisque, comme c'est le cas dans la grande majorité des mises à jour, la structure de base de l'application est inchangée. Ce serait beaucoup plus efficace si nous pouvions passer directement à l'étape 3 :

if (changed.name) {
    text.data = name;
}
(C'est presque exactement le code de mise à jour généré par Svelte. Contrairement aux frameworks d'interface utilisateur traditionnels, Svelte est un compilateur qui sait au moment de la construction comment les choses pourraient changer dans votre application, plutôt que d'attendre pour faire le travail au moment de l'exécution .)

Ce n'est pas seulement la différence
Les différents algorithmes utilisés par React et d'autres frameworks DOM virtuels sont rapides. Sans doute, la plus grande surcharge est dans les composants eux-mêmes. Vous n'écririez pas un code comme celui-ci...

function StrawManComponent(props) {
    const value = expensivelyCalculateValue(props.foo);

    return (
        <p>the value is {value}</p>
    );
}
... parce que vous recalculerez négligemment valueà chaque mise à jour, qu'elle props.fooait ou non changé. Mais il est extrêmement courant de faire des calculs et des allocations inutiles d'une manière qui semble beaucoup plus bénigne :

function MoreRealisticComponent(props) {
    const [selected, setSelected] = useState(null);

    return (
        <div>
            <p>Selected {selected ? selected.name : 'nothing'}</p>

            <ul>
                {props.items.map(item =>
                    <li>
                        <button onClick={() => setSelected(item)}>
                            {item.name}
                        </button>
                    </li>
                )}
            </ul>
        </div>
    );
}
Ici, nous générons un nouveau tableau d' <li>éléments virtuels - chacun avec son propre gestionnaire d'événements en ligne - à chaque changement d'état, qu'il props.itemsait ou non changé. À moins que vous ne soyez obsédé par la performance, vous n'allez pas l'optimiser. Il est inutile. C'est assez rapide. Mais vous savez ce qui serait encore plus rapide ? Ne pas faire ça.

React Hooks redouble d'efforts pour ne pas effectuer de travail inutile, avec des résultats prévisibles .

Le danger de ne pas effectuer un travail inutile, même si ce travail est trivial, est que votre application finira par succomber à la "mort par mille coupes" sans goulot d'étranglement clair pour viser une fois qu'il est temps d'optimiser.

Svelte est explicitement conçu pour vous empêcher de vous retrouver dans cette situation.

Pourquoi les frameworks utilisent-ils alors le DOM virtuel ?
Il est important de comprendre que le DOM virtuel n'est pas une fonctionnalité . C'est un moyen d'atteindre une fin, la fin étant un développement d'interface utilisateur déclaratif et piloté par l'état. Le DOM virtuel est précieux car il vous permet de créer des applications sans penser aux transitions d'état, avec des performances généralement suffisantes . Cela signifie moins de code bogué et plus de temps passé sur des tâches créatives au lieu de tâches fastidieuses.

Mais il s'avère que nous pouvons obtenir un modèle de programmation similaire sans utiliser de DOM virtuel - et c'est là qu'intervient Svelte.