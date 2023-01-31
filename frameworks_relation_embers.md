MISE EN RELATION :


Ember Data inclut plusieurs types de relations intégrés pour vous aider à définir les relations entre vos modèles.

Un par un
Pour déclarer une relation un-à-un entre deux modèles, utilisez belongsTo:

app/models/user.js
import Model, { belongsTo } from '@ember-data/model';

export default class UserModel extends Model {
  @belongsTo('profile') profile;
}
app/models/profile.js
import Model, { belongsTo } from '@ember-data/model';

export default class ProfileModel extends Model {
  @belongsTo('user') user;
}
Un à plusieurs
Pour déclarer une relation un-à-plusieurs entre deux modèles, utilisez belongsToen combinaison avec hasMany, comme ceci :

app/models/blog-post.js
import Model, { hasMany } from '@ember-data/model';

export default class BlogPostModel extends Model {
  @hasMany('comment') comments;
}
app/models/comment.js
import Model, { belongsTo } from '@ember-data/model';

export default class CommentModel extends Model {
  @belongsTo('blog-post') blogPost;
}
Plusieurs à plusieurs
Pour déclarer une relation plusieurs-à-plusieurs entre deux modèles, utilisez hasMany:

app/models/blog-post.js
import Model, { hasMany } from '@ember-data/model';

export default class BlogPostModel extends Model {
  @hasMany('tag') tags;
}
app/models/tag.js
import Model, { hasMany } from '@ember-data/model';

export default class TagModel extends Model {
  @hasMany('blog-post') blogPosts;
}
Inverses explicites
Ember Data fera de son mieux pour découvrir quelles relations correspondent les unes aux autres. Dans le code un-à-plusieurs ci-dessus, par exemple, Ember Data peut comprendre que la modification de la commentsrelation devrait mettre à jour la blogPost relation à l'inverse car blogPostc'est la seule relation avec ce modèle.

Cependant, vous pouvez parfois avoir plusieurs belongsTo/ hasManys pour le même type. Vous pouvez spécifier quelle propriété du modèle associé est l'inverse à l'aide de l' option belongsToou . Les relations sans inverse peuvent être indiquées comme telles en incluant .hasManyinverse{ inverse: null }

app/models/comment.js
import Model, { belongsTo } from '@ember-data/model';

export default class CommentModel extends Model {
  @belongsTo('blog-post', { inverse: null }) onePost;
  @belongsTo('blog-post') twoPost;
  @belongsTo('blog-post') redPost;
  @belongsTo('blog-post') bluePost;
}
app/models/blog-post.js
import Model, { hasMany } from '@ember-data/model';

export default class BlogPostModel extends Model {
  @hasMany('comment', {
    inverse: 'redPost'
  })
  comments;
}
Relations réflexives
Lorsque vous souhaitez définir une relation réflexive (un modèle qui a une relation avec lui-même), vous devez définir explicitement la relation inverse. S'il n'y a pas de relation inverse, vous pouvez définir l'inverse sur null.

Voici un exemple de relation réflexive un-à-plusieurs :

app/models/dossier.js
import Model, { belongsTo, hasMany } from '@ember-data/model';

export default class FolderModel extends Model {
  @hasMany('folder', { inverse: 'parent' }) children;
  @belongsTo('folder', { inverse: 'children' }) parent;
}
Voici un exemple de relation réflexive un-à-un :

app/models/user.js
import Model, { attr, belongsTo } from '@ember-data/model';

export default class UserModel extends Model {
  @attr('string') name;
  @belongsTo('user', { inverse: 'bestFriend' }) bestFriend;
}
Vous pouvez également définir une relation réflexive qui n'a pas d'inverse :

app/models/dossier.js
import Model, { belongsTo } from '@ember-data/model';

export default class FolderModel extends Model {
  @belongsTo('folder', { inverse: null }) parent;
}
Polymorphisme
Le polymorphisme est un concept puissant qui permet à un développeur d'abstraire des fonctionnalités communes dans une classe de base. Prenons l'exemple suivant : un utilisateur avec plusieurs méthodes de paiement. Ils pourraient avoir un compte PayPal lié et quelques cartes de crédit enregistrées.

Notez que, pour que le polymorphisme fonctionne, Ember Data attend une déclaration "type" de type polymorphe via la type propriété réservée sur le modèle. Confus? Voir la réponse de l'API ci-dessous.

Examinons d'abord les définitions du modèle :

app/models/user.js
import Model, { hasMany } from '@ember-data/model';

export default class UserModel extends Model {
  @hasMany('payment-method', { polymorphic: true }) paymentMethods;
}
app/models/payment-method.js
import Model, { belongsTo } from '@ember-data/model';

export default class PaymentMethodModel extends Model {
  @belongsTo('user', { inverse: 'paymentMethods' }) user;
}
app/models/payment-method-cc.js
import { attr } from '@ember-data/model';
import PaymentMethod from './payment-method';

export default class PaymentMethodCcModel extends PaymentMethod {
  @attr last4;

  get obfuscatedIdentifier() {
    return `**** **** **** ${this.last4}`;
  }
}
app/models/payment-method-paypal.js
import { attr } from '@ember-data/model';
import PaymentMethod from './payment-method'

export default class PaymentMethodPaypalModel extends PaymentMethod {
  @attr linkedEmail;

  get obfuscatedIdentifier() {
    let last5 = this.linkedEmail
      .split('')
      .reverse()
      .slice(0, 5)
      .reverse()
      .join('');

    return `••••${last5}`;
  }
}
Et notre API pourrait configurer ces relations comme suit :

{
  "data": {
    "id": "8675309",
    "type": "user",
    "attributes": {
      "name": "Anfanie Farmeo"
    },
    "relationships": {
      "payment-methods": {
        "data": [
          {
            "id": "1",
            "type": "payment-method-paypal"
          },
          {
            "id": "2",
            "type": "payment-method-cc"
          },
          {
            "id": "3",
            "type": "payment-method-apple-pay"
          }
        ]
      }
    }
  },
  "included": [
    {
      "id": "1",
      "type": "payment-method-paypal",
      "attributes": {
        "linked-email": "ryan@gosling.io"
      }
    },
    {
      "id": "2",
      "type": "payment-method-cc",
      "attributes": {
        "last4": "1335"
      }
    },
    {
      "id": "3",
      "type": "payment-method-apple-pay",
      "attributes": {
        "last4": "5513"
      }
    }
  ]
}
Données imbriquées en lecture seule
Certains modèles peuvent avoir des propriétés qui sont des objets profondément imbriqués de données en lecture seule. La solution naïve serait de définir des modèles pour chaque objet imbriqué et d'utiliser hasManyet belongsTode recréer la relation imbriquée. Cependant, étant donné que les données en lecture seule n'auront jamais besoin d'être mises à jour et enregistrées, cela entraîne souvent la création d'une grande quantité de code pour très peu d'avantages. Une autre approche consiste à définir ces relations à l'aide d'un attribut sans transformation ( @attr). Cela facilite l'accès aux valeurs en lecture seule dans d'autres objets et modèles sans la surcharge liée à la définition de modèles superflus.

Création d'enregistrements
Supposons que nous ayons a blog-postet un commentmodèle. Un seul article de blog peut avoir plusieurs commentaires qui lui sont liés. La relation correcte est illustrée ci-dessous :

app/models/blog-post.js
import Model, { hasMany } from '@ember-data/model';

export default class BlogPostModel extends Model {
  @hasMany('comment') comments;
}
app/models/comment.js
import Model, { belongsTo } from '@ember-data/model';

export default class CommentModel extends Model {
  @belongsTo('blog-post') blogPost;
}
Supposons maintenant que nous voulions ajouter des commentaires à un article de blog existant. Nous pouvons le faire de deux manières, mais pour les deux, nous devons d'abord rechercher un article de blog qui est déjà chargé dans le magasin, en utilisant son identifiant :

let myBlogPost = this.store.peekRecord('blog-post', 1);
Maintenant, nous pouvons soit définir la belongsTorelation dans notre nouveau commentaire, soit mettre à jour la hasManyrelation du blogPost. Comme vous pouvez le constater, nous n'avons pas besoin de définir à la fois hasManyet belongsTopour un record. Ember Data le fera pour nous.

Voyons d'abord comment définir la belongsTorelation dans notre nouveau commentaire :

let comment = this.store.createRecord('comment', {
  blogPost: myBlogPost
});
comment.save();
Dans l'extrait ci-dessus, nous avons fait référence myBlogPostlors de la création de l'enregistrement. Cela permettra à Ember de savoir que le commentaire nouvellement créé appartient à myBlogPost. Cela créera un nouvel commentenregistrement et l'enregistrera sur le serveur. Ember Data sera également mis à jour myBlogPostpour inclure notre commentaire nouvellement créé dans sa commentsrelation.

La deuxième façon de faire la même chose est de lier les deux enregistrements en mettant à jour la relation du blogPost hasManycomme indiqué ci-dessous :

let comment = this.store.createRecord('comment', {});
let comments = await myBlogPost.comments;
comments.pushObject(comment);
comment.save().then(function() {
  myBlogPost.save();
});
Dans ce cas ci-dessus, la relation du nouveau commentaire belongsTosera automatiquement définie sur le parent blogPost.

Bien que createRecordce soit assez simple, la seule chose à laquelle il faut faire attention est que vous ne pouvez pas actuellement attribuer une promesse en tant que relation.

Par exemple, si vous souhaitez définir la authorpropriété d'un blogPost, cela ne fonctionnera pas si l' useridentifiant with n'est pas déjà chargé dans la boutique :

this.store.createRecord('blog-post', {
  title: 'Rails is Omakase',
  body: 'Lorem ipsum',
  author: this.store.findRecord('user', 1)
});
Cependant, vous pouvez facilement définir la relation une fois la promesse remplie :

let blogPost = this.store.createRecord('blog-post', {
  title: 'Rails is Omakase',
  body: 'Lorem ipsum'
});

this.store.findRecord('user', 1).then(function(user) {
  blogPost.author = user;
});
Récupération des enregistrements associés
Lorsque vous demandez des données au serveur pour un modèle qui a des relations avec un ou plusieurs autres, vous pouvez souhaiter récupérer les enregistrements correspondant à ces modèles associés en même temps. Par exemple, lors de la récupération d'un article de blog, vous devrez peut-être également accéder aux commentaires associés à l'article. La spécification JSON:API permet aux serveurs d'accepter un paramètre de requête avec la clé includeen tant que demande pour inclure ces enregistrements associés dans la réponse renvoyée au client. La valeur du paramètre doit être une liste séparée par des virgules des noms des relations requises.

Si vous utilisez un adaptateur prenant en charge JSON:API, tel que la valeur par défaut d'Ember JSONAPIAdapter, vous pouvez facilement ajouter le includeparamètre aux requêtes de serveur créées par les méthodes , findRecord()et findAll(). query()queryRecord()

findRecord()et findAll()chacun prend un optionsargument dans lequel vous pouvez spécifier le includeparamètre. Par exemple, étant donné un postmodèle qui a une hasManyrelation avec un commentmodèle, lors de la récupération d'un article spécifique, nous pouvons demander au serveur de renvoyer également les commentaires de cet article comme suit :

app/routes/post.js
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class PostRoute extends Route {
  @service store;
  model(params) {
    return this.store.findRecord('post', params.post_id, {
      include: 'comments'
    });
  }
}
Les commentaires de la publication seraient alors disponibles dans votre modèle au format model.comments.

Les relations imbriquées peuvent être spécifiées dans le includeparamètre sous la forme d'une séquence de noms de relation séparés par des points. Donc, pour demander à la fois les commentaires de la publication et les auteurs de ces commentaires, la demande ressemblerait à ceci :

app/routes/post.js
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class PostRoute extends Route {
  @service store;
  model(params) {
    return this.store.findRecord('post', params.post_id, {
      include: 'comments,comments.author'
    });
  }
}
Les méthodes et prennent chacune un query()argument sérialisé directement dans la chaîne de requête de l'URL et le paramètre peut faire partie de cet argument. Par example:queryRecord()queryinclude

app/routes/adele.js
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class AdeleRoute extends Route {
  @service store;
  model() {
    // GET to /artists?filter[name]=Adele&include=albums
    return this.store
      .query('artist', {
        filter: { name: 'Adele' },
        include: 'albums'
      })
      .then(function(artists) {
        return artists.firstObject;
      });
  }
}
Mise à jour des enregistrements existants
Parfois, nous voulons établir des relations sur des enregistrements déjà existants. Nous pouvons simplement définir une belongsTorelation :

let blogPost = this.store.peekRecord('blog-post', 1);
let comment = this.store.peekRecord('comment', 1);
comment.blogPost = blogPost;
comment.save();
Alternativement, nous pourrions mettre à jour la hasManyrelation en poussant un enregistrement dans la relation :

let blogPost = this.store.peekRecord('blog-post', 1);
let comment = this.store.peekRecord('comment', 1);
let comments = await blogPost.comments;
comments.pushObject(comment);
blogPost.save();
Suppression de relations
Pour supprimer une belongsTorelation, nous pouvons la définir sur null, ce qui la supprimera également du hasManycôté :

let comment = this.store.peekRecord('comment', 1);
comment.blogPost = null;
comment.save();
Il est également possible de supprimer un enregistrement d'une hasManyrelation :

let blogPost = this.store.peekRecord('blog-post', 1);
let comment = this.store.peekRecord('comment', 1);
let comments = await blogPost.comments;
comments.removeObject(comment);
blogPost.save();
Comme dans les exemples précédents, la belongsTorelation du commentaire sera également effacée par Ember Data.

Les relations comme promesses
Tout en travaillant avec des relations, il est important de se rappeler qu'elles renvoient des promesses.

Par exemple, si nous devions travailler sur les commentaires asynchrones d'un blogPost, il faudrait attendre que la promesse soit tenue :

let blogPost = this.store.peekRecord('blog-post', 1);

let comments = await blogPost.comments;
// now we can work with the comments
Il en va de même pour les belongsTorelations :

let comment = this.store.peekRecord('comment', 1);

let blogPost = await comment.blogPost;
// the blogPost is available here
Les modèles de guidons seront automatiquement mis à jour pour refléter une promesse résolue. Nous pouvons afficher une liste de commentaires dans un blogPost comme ceci :

<ul>
  {{#each this.blogPost.comments as |comment|}}
    <li>{{comment.id}}</li>
  {{/each}}
</ul>

Ember Data interrogera le serveur pour les enregistrements appropriés et restituera le modèle une fois les données reçues.