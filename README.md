# recipe-amplify

# Set up
1. In command line install aws-amplify
```
npm install -g @aws-amplify/cli
```

2. In command line, use Expo to create a project called `recipe-amplify`
```
npm install -g expo-cli
expo init recipe-amplify
cd recipe-amplify
```
3. Set up amplify credentials. 
First, open default web browser and log into your aws console with your email address.

```
amplify configure
```

When prompted, you can choose us-east-1, but remember the location you chose.
specify the name of the IAM user: `recipe-amplify`

Conitnue with the settup and clicke "Create user", download the csv file. 

Go back to the command line, and enter the access key and secret key shown on screen. 

For profile name, put `recipe-amplify`

4. Make sure you are now in the `recipe-amplify` directory
```
amplify init
```
Use `recipe-amplify` as the name of the project
Choose default for everything except for profile, click the one we just created (`recipe-amplify`)



5. Make sure you downloaded xcode if you want to see IOS stimulator, and download Android stutio if you want to see Android stimulator.

```
expo start
```
Then open IOS stimulator to see the default screen.
Run the command above to see whether everything was set up right. 


6. Create graphQL API to store our data
```
amplify add api
```

select GraphQL

Provide API name: `recipes`
Choose an authorization type for the API: API key
Do you have an annotated GraphQL schema? No
Do you want a guided schema creation? Yes
What best describes your project: Single object with fields (e.g., “Todo” with ID, name, description)
Do you want to edit the schema now? Yes


In the schema.graphql file, we can decide what our datastore structure look like. You can find the file in `./src/amplify/backend`

We will change it to this:

```

type Item @model {
  id: String!
  name:String
  description:String
  tag: [String]
  rating: Float
  image: String
  ingredient:[IngredientItem]
  ingredientGroup:[IngredientGroupItem]
  step:[StepItem]
  notes:String
  forked:String
}

type IngredientItem {
  name:String
  amount:String
  unit:String
  preparation:String

}

type IngredientGroupItem {
  name:String
  ingredient:[IngredientItem]
}

type StepItem {
  description:String
}
```

We can always come back and modify it later.


Now go back to command line and run `amplify push` for aws to generate our datastore and push the information to the cloud

It will take a few minutes. 

7. install a few more dependecies
- npm install @react-native-community
- npm install @react-native-community/netinfo
- npm install aws-appsync graphql-tag
- npm install @react-native-async-storage/async-storage

(The last dependency allows you to make mutations, queries to the aws datastore using function calls)

# Connect to AWS Client Datastore using AWS AppSync

```javascript
import Amplify from '@aws-amplify/core';
import config from './src/aws-exports';
Amplify.configure(config);

import * as mutations from './src/graphql/mutations';
import * as queries from './src/graphql/queries';
import AWSAppSyncClient, { AUTH_TYPE } from 'aws-appsync';
import gql from 'graphql-tag';
import { listItems } from './src/graphql/queries';

import React from "react";
import {Image,ActivityIndicator,Alert,SafeAreaView,ScrollView,FlatList,Button,
  TouchableOpacity,StyleSheet,View,Text,StatusBar} from "react-native";
import { Ionicons } from '@expo/vector-icons';

import recipe from "./recipe.json'

```

The recipe.json file can be found here:

```javascript
const client = new AWSAppSyncClient({
  url: config.aws_appsync_graphqlEndpoint,
  region: config.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.API_KEY, // or type: awsconfig.aws_appsync_authenticationType,
    apiKey: config.aws_appsync_apiKey,
  },
  disableOffline:true,
});


```

Let's try add all the recipe Object into the aws datastore. 

```javascript
class HomeFlatlist extends React.Component {

 constructor(props) {
    super(props)
    this.state = {
      loadingDatastore: false,
      loadingAsyncStorage:false,
      datastore: [],
      saved:[],
      savedKeys:[],
      };
    };
    
    ComponentDidMount() {
    this.addRecipes();
}
    
     addRecipes = async () => {
        this.setState({loadingDatastore:true});
        for (const [index,value] of recipe.entries()) {
          try
        {const newItem = await API.graphql({ query: mutations.createItem, variables: {input: value}});
        console.log("success:  "+value.id);
        }
        catch (e) {console.log(e)}
        }
        this.setState({loadingDatastore:false});
      };
      
      
      
      
      render() {
      if (this.state.loadingDatasotre) { //if loading not finished yet, simply return a activity indicator
      return (
      <ActivityIndicator size="large"/>)
      }
      return (
      
     
      <Text>Added all recipes to datastore!</Text>
     )
      }
      
}
export default class App extend React.Component  {


render() {

<SafeAreaView>
<HomeFlatlist/>
</SafeAreaView>
}
}
```

Then we can run `npm run ios` again, notice the output in the terminal. 

We can go to our AppSync[https://console.aws.amazon.com/appsync/home?region=us-east-1#/apis], and click the one we just created APIS->recipeamplify-dev -> Data Source -> Resource.

Click on the link to be directed to dynamoDB and you can look at the items we just created. 

 Now everything is on cloud, we can fetch from cloud when we need them. 

## Create another function inside the `HomeFlatlist` class called fetchDatastore.

What this function will do is to fetch all the records from the datastore and compare with the locally saved (to be implemented with Async Storage) to determine the boolean `favorited` attribute. If the recipe exists in async storage, then `favorited` should be `true`, else it should be `false`. 

### Update the ComponentDidMount

```javascript
//these should be inside the HomeFlatlist class
ComponentDidMount() {

this.fetchDatastore();

}
 fetchDatastore = async () => { 
      this.setState({loadingDatastore:true});
      try {
      client.query({
        query: gql(listItems)
      }).then(({ data: { listItems } }) => {
        this.setState({datastore:
          listItems.items.map(
            item => {
              item.favorited = this.state.savedKeys.includes(item.id);
            return item;}
          )
        });
      }).then(data=>console.log(data));
    } catch (e) {
      console.log(e);
    } 
    
    this.setState({loadingDatastore:false});  

    };
      
```

You can run `npm run ios` to see the outputed message from the terminal. 

# AsyncStorage
First we need to import the libarary.

Then we need to create some basic function calls we can use to fetch, update and delete every time we favorite an item. 
```javascript
import AsyncStorage from "@react-native-async-storage/async-storage"

...
//these functions can be outside of the classes
async function removeObject(key) {
  try {
    await AsyncStorage.removeItem(key)
  } catch(e) {
    // remove error
  }

  console.log('removed: ' + key)
}
async function getMyObject(key) {
  try {
    
    const jsonValue = await AsyncStorage.getItem(key);
    return jsonValue != null ? jsonValue : false
  } catch(e) {
    // read error
  }

  console.log('Retrieved: '+key)

}

async function setObjectValue(key,value) {
  try {
    const jsonValue = JSON.stringify(value)
    await AsyncStorage.setItem(key, jsonValue)
  } catch(e) {
    // save error
  }

  console.log('Stored.')
}
```

Next, let't create a function to fetch all the local store (AsyncStorage) and set everything in the `saved` props. All the saved keys will be stored in `savedKeys` prop. 

We will update the ComponentDidMount() again. This time, we will await `this.fetchAsyncStore()` before executing `this.fetchDatastore()`. We want to make sure when we compare the aws datastore with `savedKeys` prop (to determine `favorited` attribute of every item), the `savedKeys` are already loaded. 



```javascript

//inside the HomeFlatlist class

ComponentDidMount() {

this.fetchAsyncStore().then(this.fetchDatastore());
}
fetchAsyncStore = async() => {
      this.setState({loadingAsyncStorage:true});
      try {
        const keys = await AsyncStorage.getAllKeys();
        const result = await AsyncStorage.multiGet(keys);
        let currAsyncStore = result.map(req => {
          var key = req[0];
          var value = JSON.parse(req[1]);
          return (value)
        });
        this.setState({savedKeys:keys})
        this.setState({saved:currAsyncStore})
        console.log(currAsyncStore);

    
      } catch (error) {
        console.error(error)
      }
      this.setState({loadingAsyncStorage:false})
    };

```
### Flatlist
Flatlist is a scrollable react native component. It needs to have at least 3 attributes: `data`, `renderItem`,and `keyExtractor`

We will call `renderItem` function to create a Component for each item inside the `data`. `keyExtractor` will be usde to determine the key of each Component we will be rendering. 


```javascript

//inside the HomeFlatlist Class
renderItem = data => {

<View style={{flexDirection:"row"}}/>

<View className="recipe-name" style={{flex:2}}>
  <Text>{data.item.name}</Text>
</View>

<View className='favorited' style={{flex:1}}>
{data.item.favorited?"Favorited":"Not Favorited"}
</View>

<View className="favorite-action" style={{flex:1}}>
<TouchableOpacity onPress={()=>Alert.alert("Trying to favorite this item")}>
<Ionicons name="heart" size={15}></Ionicons>
</TouchableOpacity>

</View>
<View>

//we will also update the render() part 

render() {

if (this.state.loadingDatastore||this.state.loadingAsyncStorage) { //if we are still loading, return the loading indicator
return (
<ActivityIndicator size="large"/>)

} 
return (

<Flatlist
data={this.state.datastore}
keyExtractor={item=>item.id}
renderItem={item=>this.renderItem(item)}
/>

)

}

}

```

Now we can run `npm run ios` again to see a list of recipes, all of them are not favorited (because we have not enabled buttons to favorite them)


# Updating AsyncStorage

We will see that the each rendered item of the flatlist will have three parts: its name, its favorite state, and a button to favorite it (not quite implemented right). 
We want to combine the last two parts: a heart button that will indicate if it is favorited and also when we press it, the recipe will be favorited or unfavorited depending on its current state.

### Logic of Heart Button
Clear heart -> favorited:false -> Press it -> set favorited:true, update datastore, store item to local storage (AKA AsyncStorage)
Red heart -> favorited:true -> Press it -> set favorited:false, update datastore, delete item from local storage

So the button component will look something like this
This is just the logic, not actual code

```javascript
<TouchableOpacity onPress={ if favorited===false,set favorited to false, store item to Async storage;
                              if favorited===true, set favorited to true, delete item from Async storage}>

<Ionicons name="heart" size={15} color={data.item.favorited?"red":"white"></Ionicons>

</TouchableOpacity>

```


With that being said, let's create our function `selectItem`, which will replace the logic in the `onPress`. 

Because we will update the current item's `favorited` attribute, we need to update the state prop `datastore` each time we do it. 

We will first modify on `this.state.datastore`, then we will call `this.setState({datastore:this.state.datastore})` to update it. 

```javascript
selectItem = async(data) => {


      data.item.favorited = !data.item.favorited;
      if (!data.item.favorited) {
        await removeObject(data.item.id);
      } else {
        await setObjectValue(data.item.id,data.item);
      }
      const index = this.state.datastore.findIndex(
       item => data.item.id === item.id
      );
      
      this.state.datastore[index] = data.item;
      
      this.setState({
       datastore: this.state.datastore,
      });
      };

```
Now let's re-write the button's code:


```javascript

<TouchableOpacity style={{flex:1}} onPress={()=>this.selectItem(data)}> 
        <Ionicons name="heart" size={15} 
        color={data.item.favorited?"red":"white"}></Ionicons>
</TouchableOpacity>

```







