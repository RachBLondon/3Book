# 3Book - A 3Box Tutorial

This tutorial takes your through the steps of working with 3Box spaces and the profile hover plugin.


## Setup and Tech Stack

For the tutorial, some familiarity with react is required. You can complete by copying and pasting each step, but to get a more meaningful understanding, it is recommended you are familiar with the basics.

- **React** — frontend framework
- **[IPFS](https://ipfs.io/)** + **[OrbitDB](https://orbitdb.org/)** — where the data is stored (provided by 3Box, so we won’t need to touch this directly)
- **[MetaMask](https://metamask.io/)** — Web3 wallet integration (required to facilitate signing and encryption of data)
- **[3Box.js](https://docs.3box.io/build/web-apps)** — 3Box SDK that connects wallets to IPFS database storage via 3ID
- **[Profile Hover](https://docs.3box.io/build/plugins/profile-hover)**, and **[Profile Edit Plugins](https://docs.3box.io/build/plugins/profile-edit)** — drop-in React components that we will use to speed up UI development
- React Bootstrap - UI Library

## 0.Install Boiler Plate

- Enables a MetaMask web3 provider
- Sets up basic routing for `/` , `/notes` and `/profile` using react router
- Only a template,  **3Box**  functionality has not been added yet

## 1.Profile Hover

This is this the most minimal way to add 3Box to a project. 

**1.1** Install via npm

`npm i profile-hover`

Then import to the top of `App.js`

    import ProfileHover from 'profile-hover';

**1.2**  We are going to use this component  to replace where we are rendering the users **Ethereum address** in the **`Home`** component.  Finding 

    <h2>{this.props.ethAddress}</h2>

We are going to replace it with 

     <ProfileHover address={this.props.ethAddress} showName={true}/>

After saving you should now see something like this 👇

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/af5cc32d-4d6e-437e-8aa7-cd3066612dd9/profile-hover.gif](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/af5cc32d-4d6e-437e-8aa7-cd3066612dd9/profile-hover.gif)

The **Profile Hover** component is uses data and images set to a user's 3Box Profile. This can be changed in **[3Box Hub](https://3box.io/hub).** Later on in this tutorial we will learn how to us the **Profile Edit component** to update directly from your application.

## 2.Storage

Storing data in a user controlled way on top of **IPFS** and **OrbitDB** is one of the of the core functionalities of 3Box. **3Box Spaces** lets users store data in **public** (unencrypted) or **private** (encrypted) form in a spaces with cryptographic access control. In this tutorial we are going to make use of the **Spaces API** to provide personal note taking functionality.

Before we can work with **Spaces** we need to install and authenticate with 3Box.

### **2.1 3Box Installation and Authentication**

We will start by installing ****and importing **3Box**

`npm i 3box`

    import Box from '3box'; 

We import as "Box" rather than "3Box" as variables that begin with numbers are not valid in JavaScript.

Next I created the following method for the `App` component

    async auth3box() {
        const address = this.state.accounts[0];
        const spaces = ['3Book'];
        const box = await Box.create(window.ethereum);
        await box.auth(spaces, { address });
        await box.syncDone;
        this.setState({ box });
      }

**Create a 3Box Instance**

This function calls the `Box.create` method. This takes an **Ethereum provider** (in this case `window.ethereum`  - if you are using a wallet different to MetaMask this could differ) and creates a **3Box** instance, which can then be used to **authenticate the user** (or to open threads).

**Authenticate and sync 3Box**

Calling `box.auth` on the **3Box instance** will authenticate your user. Here you pass in an array of **spaces** (storage areas) and the users **Ethereum address.** The Ethereum address was obtained in the the boilerplate code, when MetaMask was enabled. See `getAddressFromMetaMask`.

Finally after `Box.create` and `box.auth` , we sync our authenticated 3Box instance with `box.syncDone` and add it to react state.

**Calling auth3Box**
We can call the `auth3Box` method inside `componentDidMount`, after we have be granted access to the users **Ethereum address**

    async componentDidMount() {
        await this.getAddressFromMetaMask();
        if (this.state.accounts) {
          // Now MetaMask's provider has been enabled, we can start working with 3Box
    		
    		this.auth3Box();
        }
      }

🚨Running the app after installing **3Box** may result in the following error:

`Module not found: Can't resolve 'multicodec/src/name-table'`

This is due to an issue with the version of IPFS we run - [we are working on fixing this by upgrading to the latest IPFS version.](https://github.com/3box/3box-js/issues/687)  A temporary solution for this is to install multicodec directly:

`npm i multicodec@0.5.6`

### 2.2 Opening a Space

The next thing we want to do is open a space the space we authenticated in the step above. Spaces are where used to store data in 3Box. For more information visit our **[Architecture blog post](https://medium.com/3box/3box-architecture-a3e35c82e919).** In this tutorial we will be using a space to store notes our users make.

After we call our `auth3box` method, we will add the `box.openSpace` and `space.syncDone` methods. After opening our space, we have full functionality to write and read from it. Then we will save our space to **react state** for it to accessible throughout our app.

    async componentDidMount() {
        await this.getAddressFromMetaMask();
        if (this.state.accounts) {
          await this.auth3box();
    
          const space = await this.state.box.openSpace('3Book');
          await space.syncDone;
    			this.setState({space});
        }
      }

### 2.3 Working with public and private spaces

Everything is ready for us to work with spaces now. Here we will be creating the functionality for our users to store **public** (unencrypted) and **private** (encrypted) notes. This will be handled by the `<Notes/>` component. We already have the template for the form for users to submit their notes.

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/aa2b9ddf-cb9e-4b88-9d66-0e19e9eb28f5/Screenshot_2020-01-24_at_13.07.46.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/aa2b9ddf-cb9e-4b88-9d66-0e19e9eb28f5/Screenshot_2020-01-24_at_13.07.46.png)

The view button here will toggle between viewing and submitting notes. As we won't have any notes in our space yet (it's new). Let's start with the functionality of adding notes.

Our template does not yet save any data to the 3Box network. We have methods for `publicSave` and `privateSave` , so far they are only handling saving and removing from the react state.

Let's add to them the following code:

    publicSave = async (e) => {
        e.preventDefault();
    	
    		//saves to a public 3Box space
        await this.props.space.public.set(Date.now(), this.state.publicNoteToSave);
    		
    		// Clears up UI after save
    		this.setState({publicNoteToSave : null});
        console.log("saved")
      }
    
      privateSave = async (e) => {
        e.preventDefault();
        
    		//saves to a private 3Box space
    		await this.props.space.private.set(Date.now(), this.state.privateNoteToSave);
    		
    		// Clears up UI after save
    		this.setState({privateNoteToSave : null});
        console.log("saved");
      }

These is where we are save the form data to the users **space** in either **private** or **public** form with the `space.public.set` and `space.private.set` functions. These functions take two arguments, a key and a value. The key can later be used to retrieve that specific value. In this case we are just using a timestamp as a key. [Read more about saving to a space in our docs](https://docs.3box.io/api/storage/set). After setting this data we can remove the data from same data from the **react state.**

Next we need to make sure we pass in the **space** we opened in step 2.2, along as a property to the `Notes` component:

    <Notes space={this.state.space}/>

We should add another condition to the conditional which is wrapping the two `FormComponent` . We only want these components to render once we can access `this.props.space` otherwise we will encounter errors.

    {!this.state.view && this.props.space && (<>
              <h3>📖Public</h3>
              <FormComponent
                handleSubmit={this.publicSave}
                onChange={(e)=>(this.setState({publicNoteToSave : e.target.value}))}
                value={this.state.publicNoteToSave}
                label="Save a Public Note"
                text="This text will be saved publicly on 3Box"
              />
              <br />
    
              <h3>🗝Private</h3>
              <FormComponent
                handleSubmit={this.privateSave}
                onChange={(e)=>(this.setState({privateNoteToSave : e.target.value}))}
                value={this.state.privateNoteToSave}
                label="Save a Private Note"
                text="This text will be encrypted and saved with 3Box"
              />
            </>
    )}

### 2.4 Getting Data from Spaces

Now we can set data to a **space**, the next step its to retrieve data from our space, and show it in the UI. Let's add these two methods to our `Notes` component:

    getPublicNotes = async () => {
        const publicNotes = await this.props.space.public.all();
        this.setState({ publicNotes });
      }
    
      getPrivateNotes = async () => {
        const privateNotes = await this.props.space.private.all();
        this.setState({ privateNotes });
      }

Here we calling [**space.public.all**](https://docs.3box.io/api/profiles/get#box-public-all-opts) and **[space.private.all](https://docs.3box.io/api/profiles/get#box-private-get-key-opts)** methods to retrieve our **public** and **private data**. In this case, we are returning all of our data in each part of our space. However if you wanted to get a specific value, you can query using the spaces **[get** **method](https://docs.3box.io/api/profiles/get#example-3)** and by passing in a value.

After we have returned the data from our space, we save it to react state. We can call both of these methods in `componentDidUpdate` in `Notes` . These methods are nested inside a **conditional.** We only want to be able to call them when we can access the **space** and we don't have either the public or private notes stored in react state.

    componentDidUpdate(){
        if(this.props.space && (!this.state.privateNotes || !this.state.publicNotes)){
          this.getPublicNotes();
          this.getPrivateNotes();
        }
      }

Then lets render the notes in the UI but updating the view section of our `Notes` component.

    {this.state.view && <>
              <h2>View</h2>
              <br />
              <h3>📖Public</h3>
              {this.state.publicNotes &&  Object.values(this.state.publicNotes).map(note => <p>{note}</p>)}
              <br />
              <h3>🗝Private</h3>
              {this.state.privateNotes && Object.values(this.state.privateNotes).map(note => <p>{note}</p>)}
            </>}

Finally we can also call these methods to get our notes, directly after we save them, so they appear in our UI straightaway.

    publicSave = async (e) => {
        e.preventDefault();
        //saves to a public 3Box space
        await this.props.space.public.set(Date.now(), this.state.publicNoteToSave);
    
        this.setState({publicNoteToSave : null});
        console.log("saved");
        this.getPublicNotes();
      }
    
      privateSave = async (e) => {
        e.preventDefault();
    
        //saves to a private 3Box space
    		await this.props.space.private.set(Date.now(), this.state.privateNoteToSave);
    
        this.setState({privateNoteToSave : null});
        console.log("saved");
        this.getPrivateNotes();
      }
