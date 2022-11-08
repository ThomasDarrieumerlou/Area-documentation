Details
========

Le projet est composer de plusieurs partie utile au bon fonctionnement du projet.

Authentification
-----------------

Nous avons mis en place un system d'Authentification sur notre projet.
Pour ce faire la personne doit renseigner uniquement un Username et un password associer.

C'est information sont stocker dans la base de donner et chaque utilisateur est associer à un token.


cela permet de personnaliser les action selon l'utilisteur.


.. _service:

Services
---------

:doc:`api`

.. _action:

Action 
-------

L'une des composantes les plus importante du projet sont les Actions.
cette dernière est une taches réaliser sur un service suivant des instruction déterminer en amont.

Voici un diagramme explicant de façons large le fonctionnement :

.. image:: images/action.png
    :width: 400


Pour ce faire une connection au service est necessaire pour le fonctionnement voir ci-joint pour plus d'information: :ref:`service`

quelque point clés pour comprendre le composant.

Tous d'abord le composant Action est lier a un trigger définie en amont pour guider l'action a faire.
sans un trigger il ne serai pas quoi faire.

Donc des trigger on été définie pour chaque action et reaction.
Un exemple de trigger :
    
    Connection au gmail
    
    check tous les x temps si nous avons reçus un mail
    
    stockage des information du mail.

pour manager tous les services nous utilision un call de variable

.. code-block:: go


    var Google = google.New()
    var Microsoft = Microsoft.New()

Par exmple pour la connection à google on va demander un Authentification au service

.. code-block:: go

    func (*googleService) Authenticate(callback string, userId uint) string {
	var state types.OauthState

	state.Callback = callback
	state.UserId = userId

	conf := &oauth2.Config{
		ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
		ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
		RedirectURL:  os.Getenv("OAUTH_REDIRECT_URL") + "/providers/google/callback",
		Scopes: []string{
			"https://www.googleapis.com/auth/gmail.readonly",
		},
		Endpoint: google.Endpoint,
	}
	bytes, _ := json.Marshal(state)
	str := base64.RawStdEncoding.EncodeToString(bytes)
	return conf.AuthCodeURL(str, oauth2.AccessTypeOffline, oauth2.ApprovalForce)
    }

on récupèr donc l'id et le secret du client pour pouvoir ensuite intéragir avec le service google pour l'action.

Lorsque l'accés au service est bon on va lui demander de faire une action qu'on a définie comme ci-dessous :abbr:

.. code-block:: go
    
    func (*googleService) Check(action string, trigger models.Trigger) bool {
	var srv = createGoogleConnection(trigger.User.GoogleToken)
	if srv == nil {
		return false
	}
	switch action {
	case "receive":
		return checkGmailReceive(srv, trigger.ID, trigger.UserID)
	case "send":
		return checkGmailSend(srv, trigger.ID, trigger.UserID)
	}
	return false
    }

Dans notre cas on peux observer qu'on effectue des tache vis a vis de l'envoie et la reception de mail.
pour réaliser une reaction en fonction de cette action nous allons par exemple stocker les informations du mail

donc nous allons récuperer les info 

.. code-block:: go
    
    func fetchLastGmailReceive(srv *gmail.Service) *gmail.Message {
	res, err := srv.Users.Messages.List("me").Q("label:Inbox").Do()
	if err != nil {
		lib.LogError(err)
		return nil
	}

	id := res.Messages[0].Id
	res2, err := srv.Users.Messages.Get("me", id).Do()
	lib.LogError(err)
	return res2
    }   

cette action est réaliser nous allons passer donc au reaction.

.. _reaction:

Reaction
---------

La reaction est la suite d'une action que nous avons effectuer en amont.
Par exemple ql'action reaction qui est de checker si un mail à ete envoyer puis nous envoyer un message sur discord

pour comprendre la logique voici un diagramme :

.. image:: images/réaction.png
    :width: 400


Nous avons vue en amont le check de reception de mail sur gmail.

Maintenant la reaction de cette action sur discord. Pour ce faire il faut c'authentifier sur discord
puis effectuer l'action "send". cela ce traduit comme ceci :

.. code-block:: go

    func (*discordService) React(reaction string, trigger models.Trigger) {
	var storedData models.TriggerData
	var buf bytes.Buffer

	buf.Write(trigger.Data)
	err := gob.NewDecoder(&buf).Decode(&storedData)
	lib.LogError(err)

	switch reaction {
	case "send":
		sendMessage(storedData, trigger.Action, trigger.ActionService)
	}
    }

Dans ce cas on lui donne l'instruction d'envoyer un message sur discord.
pour ce faire dans le cas de discord nous avons l'authorisation d'utiliser un webhook. si non cela n'est pas authoriser.

donc voici l'exemple avec un webhook :

.. code-block:: go

    func sendMessage(storedData models.TriggerData, action string, service string) {
	var username = "Area"
	var title = "New " + string(action) + " in " + string(service)
	var color = "1668818"
	var embeds []discordwebhook.Embed
	var fields []discordwebhook.Field
	var timestamp = storedData.Timestamp.Format("January 2, 2006") + " at " + storedData.Timestamp.Format("15:04:05")
	var footer discordwebhook.Footer = discordwebhook.Footer{
		Text: &timestamp,
	}

	url := storedData.ReactionData

	fields = append(fields, discordwebhook.Field{
		Name:  &storedData.Title,
		Value: &storedData.Description,
	})
	embeds = append(embeds, discordwebhook.Embed{
		Title:       &title,
		Description: &storedData.Author,
		Fields:      &fields,
		Color:       &color,
		Footer:      &footer,
	})
	message := discordwebhook.Message{
		Username: &username,
		Embeds:   &embeds,
	}
	err := discordwebhook.SendMessage(url, message)
	lib.LogError(err)
}


Mais dans une grand majoriter par exemple pour .... nous faisons comme ci-dessous :

.. code-block:: go
    
	func (jobsManager) Do() {
	var triggered bool
	triggers, err := database.Trigger.GetActive()
	lib.LogError(err)

	for _, v := range triggers {
		switch v.ActionService {
		case "google":
			triggered = services.Google.Check(v.Action, v)
		case "microsoft":
			triggered = services.Microsoft.Check(v.Action, v)
		case "github":
			triggered = services.Github.Check(v.Action, v)
		case "notion":
			triggered = services.Notion.Check(v.Action, v)
		case "discord":
			triggered = services.Discord.Check(v.Action, v)
		default:
			triggered = false
		}
		if triggered {
			updated, _ := database.Trigger.GetById(v.ID, v.UserID)
			switch v.ReactionService {
			case "google":
				services.Google.React(v.Reaction, *updated)
			case "microsoft":
				services.Microsoft.React(v.Reaction, *updated)
			case "github":
				services.Github.Check(v.Reaction, *updated)
			case "notion":
				services.Notion.React(v.Reaction, *updated)
			case "discord":
				services.Discord.React(v.Reaction, *updated)
			}
		}
	}
	}