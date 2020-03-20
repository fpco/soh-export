# Websockets Reactive Game of Score

UPDATES: (1) To play the Game of Score, go to (http://game.schalk.site[)http://game.schalk.site].. There is a link in (http://schalk.site)[http://blog.schalk.site]. Score is a multiplayer game among players in the same, user-defined group. Group players share a text box and a todo list.

(2) While the original modified Wai Websockets server, with functionality added as needed, is still the back end of David Schalk's interactive web sites, the front ends no longer rely on React. They use the less opionated and more streamlined framework [cycle.js](https://cycle.js.org). 

David Schalk's recent project, (http://schalk.site)["Functional Programming and Javascript"], running online at [http://schalk.site](http://schalk.site), uses the original modified Wai Websockets server with some additions that facilitate running interactive demonstrations. The code is at [Github](https://github.com/dschalk/functional-javascript). It has been very easy to maintain as additional features become necessary. You immediately know what side-effects-free pattern-matching functions are doing. Overall, it has been a joy to work with.

This tutorial shows how readily Wai Websockets can serve as a backend to a fairly complex interactive web application. It was surprisingly easy to make sure user actions are seen in the browsers of fellow group members and only in the browsers of fellow group members. It performed flawlessly whie it grew in order to support the game of Score. 

The front end and the back end separate Ubuntu 18.04 applications on seoarate droplets hosted by digitalocean.com in the New York area. I haven't noticed any down time, the geting-started tutorials are great, and it is easy to run Haskell code on the droplets. I currently use stack runghc to start the WebSockets server but I have run compiled Haskell binaries on droplets in the past. 

I hear there is a Google Student account that includes a digital ocean droplet. Digitalocean says "Everyone you refer gets $50 in credit over 30 days. Once they’ve spent $25 with us, you'll get $25. There is no limit to the amount of credit you can earn through referrals." if you use this link: [https://m.do.co/c/1d035c466e8d](https://m.do.co/c/1d035c466e8d)and I have no reason to doubt them.

THE HASKELL CODE:

I use TMVar instead of the original plain MVar; I don't use literate Haskell; and, of course, I added some application-specific code. The state list is defined as follows:

```haskell
type Name = Text
type Score = Int
type Group = Text
type Client = (Name, Score, Group, WS.Connection)
type ServerState = [Client]
```
It is instantiated in main with:
```haskell
newServerState :: ServerState
newServerState = []

main :: IO ()
main = do
    state <- atomically $ newTMVar newServerState
    . . .
```
This is identical to the ServerState used in JS-monads-stable, only JS-monads-stable Client has one more parameter. Explanations and demonstrations can be found at [JS-monads-stable](http://schalk.net:3055) and [Repository](https://github.com/dschalk/monads-for-javascript). 

The following sections probably provide more information than you want, but for what they are worth, here are several glimpses at various features of the Game of Score application. Some familiarity with React might make parts of the front-end code more comprehensible. I don't adhere to recommended React best practice. I recommend Cycle.js. I you just want to see the Haskell part of this presentation, just skim the front-end discussion to get an idea of what the server is doing.

##Requiring players to use the result of a prior computation.

In the game of *Score*, players roll four dice and try to make the number '20' in two or three stages using addition, subtraction, multiplication, division, and concatenation. This is a cumulative process, always using the results of prior computations. After the first computation, two of the original die remain along with the result of the computation. Now say the roll was [4,5,3,9] and the first computation was 3 + 8 = 11. The choices for the next computation are now [4,5,11]. I made the result of the first computation red and displayed a message saying the player must use the red number in order to score a point in the current round. I needed to make sure that computing 4 * 5 = 20 would not increase the player's score. A player would have to use '12' in order to score a point in this round.The player could, however, run 4 + 5 = 9, yielding [11,9], and then 11 + 9 = 20, scoring one point.

In order to assure use of a prior result, I created a state element named 'test' with an initial vlue of 'false' along with an element named 'resPrevious' to hold the result of each computation. The two numbers selected in the second round are placed in an array which is tested for the presence of 'resPrevious'. The Boolean result is assigned to 'test'. After the first computation, the code shown two code snippets down from here is run. 

DES_ws sends four messages to the server. 'gr' is the player's group, which ultimately decides which players see a scoreboard update. The if statement tests for a result of '20' (or a player-defined goal; '20' is the default); test === true, meaning the prior result was used; and 'score', which tests whether the player clicked score to start a ten-second countdown. 

The value of 'test' is set in 'calc()', the calculation function, with the following code:

```js
    let resP = this.state.resPrevious;
    let ar5 = [mes0,mes2];
    let test = (resP === mes0 || resP === mes2);
```
'mes0' and 'mes2' are the currently selected numbers. They arrive at 'calc()' as arguments. 

calc() calls a function named 'comp' with five arguments. Setting state with setState during number selection requires the use of callbacks to make sure values are timely updated. If, for example, I were to select the number held in message1 and just call 'setState({message1: ''}), when I got to comp(), this.state.message1 would still have its old value. Here is the code for selecting the left-most number:

```js
  handleB40 = () => {
    let name = this.state.name;
    let group = this.state.group;
    let num = this.state.message1;
    if (this.state.mes0 === 'Number') {
      this.setState({message1: '', mes0: num}, function() {
        DES_ws.send(`CQ#$42,${group},${name},${num}`);
      })
    }
    else if (this.state.mes2 === 'Number') {
      this.setState({message1: '', mes2: num}, function() {
        DES_ws.send(`DQ#$42,${group},${name},${num}`);
        if (this2.state.mes1 !== 'Operator') {
          this.calc(this2.state.mes0, this.state.mes1, num);
        }
      })
    }
  }
```
If a number and operator have already been selected, handleB40 calls calc():

```js
  calc (mes0,mes1,mes2) { 
    let that = this;
    let res = 0;
    let delay = this.delay;
    let n = this.state.N;
    let resP = this.props.resPrevious;
    let ar5 = [mes0,mes2];
    let test = (resP === mes0 || resP === mes2);
    switch (mes1) {
      case "+": that.comp( parseFloat(mes0) + parseFloat(mes2),mes0,mes1,mes2,test );
      break;
      case "-": that.comp( parseFloat(mes0) - parseFloat(mes2),mes0,mes1,mes2,test );
      break;
      case "*": that.comp( parseFloat(mes0) * parseFloat(mes2),mes0,mes1,mes2,test );;
      break;
      case "/": that.comp( parseFloat(mes0) / parseFloat(mes2),mes0,mes1,mes2,test );
      break;t
      case "Concat": that.comp( parseFloat(mes0+""+mes2),mes0,mes1,mes2,test );
      break;
      default : 'operator not selected';
    }
  }
  }
```
calc() calls comp():
```javascript
  comp (result,mes0,mes1,mes2,test) {
    let str = `${mes0} ${mes1} ${mes2} = ${result}`;
    this.props.resPrevious = result.toString();
    let w1 = this.state.message1;
    let w2 = this.state.message2;
    let w3 = this.state.message3;
    let w4 = this.state.message4;
    let startArray = [w1, w2, w3, w4, result];
    this.newNums(result,str,test,startArray);
  }
  ```
  And comp() calls newNums():

```js
  newNums = (result,str,test,numbers) => {
newNums (result,str,test,numbers) {
    let j = 0;
    let gr = this.state.group;
    let ar = [];
    let clock = '';
    let name = this.state.name;
    let impossibleClicker = this.state.impossibleClicker;
    let interruptClicker = this.state.interruptClicker;
    let interrupt = this.state.interrupt;
    let test2 = this.state.score || this.state.impossible;
    let goal = 1*(this.state.goal); // '1*' and '==' is technically overkill, but seems like insurance.

    for (let k in numbers) {
        if (numbers[k] !== "" && numbers[k] !== undefined) {
        ar[j] = numbers[k];
        j += 1;
      }
    }
    if (j === 3) {
      DES_ws.send(`FQ#$42,${gr},${name},${str}`);
      DES_ws.send(`CE#$42,${gr},${name},${ar[0]},${ar[1]},${ar[2]},`);
      this.setState({message: 'You must use the number with the yellow background in order to score in this round.'});
      this.mouse[2] = 'yellow';
      if (test2) {
        DES_ws.send( `CK#$42,${gr},${name},10` );
      }
      else {
        DES_ws.send( `CK#$42,${gr},${name},Did not click SCORE!` );
      }
    }
    else if (j === 2) {
      this.setState({message: ''});
      DES_ws.send(`GQ#$42,${gr},${name},${str}`);
      DES_ws.send(`CE#$42,${gr},${name},${ar[0]},${ar[1]},,`);
      if ( (result == goal) && test && test2 && !interrupt ) {
          this.setState({DS_T: -1});
          DES_ws.send( `CK#$42,${gr},${name},One point for ${name}` );
          DES_ws.send( `CR#$42,${gr},${name},${name}` );
          DES_ws.send( `CG#$42,${gr},${name},1` );
      }
      else if ( (result == goal) && test && test2 && interrupt ) {
        this.setState({DS_T: -1});
        DES_ws.send( `CK#$42,${gr},${name},One point for ${name}. Two points deducted from ${impossibleClicker}`);
        DES_ws.send( `CR#$42,${gr},${name},${name}` );
        DES_ws.send( `CG#$42,${gr},${name},1` );
        DES_ws.send( `CG#$42,${gr},${impossibleClicker},-2` );
      }
      else if (test2) {
        DES_ws.send( `CK#$42,${gr},${name},10` );
      }
      else {
        DES_ws.send( `CK#$42,${gr},${name},Did not click SCORE!` );
      }
    }
    else if (j === 1) {
      if (result === goal) {
        if (test2) {
          if (interrupt) {
            this.setState({DS_T: -1});
            DES_ws.send( `CK#$42,${gr},${name},One point for ${name}. Two points deducted from ${impossibleClicker}`);
            DES_ws.send( `CR#$42,${gr},${name},${name}` );
            DES_ws.send( `CG#$42,${gr},${name},1` );
            DES_ws.send( `CG#$42,${gr},${impossibleClicker},-2` );
          }
          else if (!interrupt) {
            DES_ws.send( `CK#$42,${gr},${name},One point for ${name}` );
            DES_ws.send( `CR#$42,${gr},${name},${name}` );
            DES_ws.send( `CG#$42,${gr},${name},1` );
          }
        }
      }

      else if ((result !== goal) && test2) {
        if (interrupt && (impossibleClicker !== interruptClicker)) {
            console.log('cow');
            DES_ws.send(`CG#$42,${gr},${impossibleClicker},1`);
            DES_ws.send(`CG#$42,${gr},${interruptClicker},-1`);
            DES_ws.send( `CK#$42,${gr},${name},The result is not 20. ${name} lost one point. One point awarded to ${impossibleClicker}.`);
        }
        else if (interrupt && (impossibleClicker === interruptClicker)) {
              DES_ws.send(`CG#$42,${gr},${impossibleClicker},-1`);
              DES_ws.send(`CH#$42,${gr},${impossibleClicker},${impossibleClicker} forfeits two points for blocking with SCORE!`);
        }
        else if (!interrupt) {
              DES_ws.send(`CG#$42,${gr},${scoreClicker},-1`);
              DES_ws.send(`CH#$42,${gr},${scoreClicker},${scoreClicker} forfeits one point. The result is not ${goal}`);
        }
      }
    }
  }

```
The for clause toward the top of newNums() does two important things. It configures an array of number for the next round containing the remaining un-used number(s) indexed sequentially from '0', along with the new number; i.e., the result of the prior computation. newNums() also determines the integer 'j', which gets smaller on each succeeding round. 'j' is the number of items in the array. Each computation uses up two numbers and creates one number (the result). If j == 3, it is too soon for a score; if j == 2 a test makes sure a point is awarded only if the computation used the result of the first; if j == 1, the round is over.

I am using js6 function syntax '= () => {' instead of '() {' in order to eliminate the need to explicitely bind functions to B2. Instead of components and HTML elements in B2's render function having the form 'myFunc = {this.myFunc.bind(this)}', with jsx function syntax it is simply 'myFunc = {this.myFunc}'.

###Rule Interlude
If a player clicks IMPOSSIBLE instead of SCORE!, a sixty-second coundown begins. If no other player in the group clicks SCORE! before the time runs out, the player who clicked IMPOSSIBLE gets one point. A one-minute 'hour glass' is used in the table-top version. If a player clickes SCORE! before the 60 seconds expires, that player gets one point and the player who clicked IMPOSSIBLE loses two points, but only if the SCORE! clicker succeeds. If the SCORE! clicker fails to compute 20 (or the player-defined goal), that player loses one point and the player who clicked IMPOSSIBLE gains one point because time ran out with no one registering a solution. The SCORE! button is available to the IMPOSSIBLE clicker, so if during the sixty-second coundown that player sees a solution that seems fairly obvious, clicking SCORE! is a way to make sure two points won't be lost due to an opponent's clicking SCORE! and succeeding. If the IMPOSSIBLE clicker clicks SCORE! and succeeds in computing the goal, one point is gained for the success and two points are lost because a solution was found before 60 seconds expired. The net result is a loss of one point. If the IMPOSSIBLE clicker clickes SCORE! and lets the 10 seconds run out, the rules discussed so far would mean the IMPOSSIBLE clicker gains one point because no solution was found and loses one point for letting the ten seconds run out. The net result would be no gain and no loss. It would be as though the player used the IMPOSSIBLE button to take back the rash clicking of IMPOSSIBLE. The ploy won't work because of the 'no blocking' rule. If the impossible clicker clicks score and does not succeed, the impossible clicker loses two points for preventing other players from clicking SCORE! and succeeding. 

###Back to the code discussion

A player might have clicked 'SCORE!' before any other player clicked anything, or the player might have clicked 'SCORE!' after someone clicked 'IMPOSSIBLE'. The Boolean state variable 'interrupt' changes from 'false' to 'true' whenever someone clicks 'IMPOSSIBLE' and reverts to 'false' during each new roll of the dice. When a player clicks IMPOSSIBLE, the 'impossibleClicker' variable takes the value of that player's name.

I haven't explained all of newNum()' functionality, or what messages going to the server do. Here's what one category of message does: When the server receives a message prefixed by 'XXXXX', it logs the message in 'log.txt'. The version shown above doesn't have any of these, but it did during the first stages of development. The other prefixes correspond to things such as updating state and broadcasting display informaiton to the browsers. Messages prefixed by 'CG#$42' and ending with 1 or -1 cause the server to update scores in the state list of tuples in the server's TMVar. 

Here is how this is accomplished. First, the prefix causes the message data to be grabbed:
```haskell
  else if "CG#$42" `T.isPrefixOf` msg
        then
            mask_ $ do
                old <- atomically $ takeTMVar state
                let new = changeScore sender extraNum old
                atomically $ putTMVar state new
                let subSt = subState sender group new
                broadcast msg subSt
                broadcast ("CB#$42," `mappend` group `mappend` ","
                    `mappend` sender `mappend` "," `mappend` T.concat (intersperse "<br>" (textState subSt))) subSt

```
'old' is the ServerState list of tuples removed from the TMVar. 'new' mirrors 'old', only with the change caused by 'changeScore'.
```haskell
changeS :: Text -> Int -> Client -> Client
changeS x y (a, b, c, d) | x == a    = (a, b+y, c, d)
                         | otherwise = (a, b, c, d)

changeScore :: Text -> Int -> ServerState -> ServerState
changeScore name k = map (changeS name k)

```
The Int argument can be 1 or -1, depending on what was chosen in 'newNums'. As you see, changeS changes the tuple corresponding to the player named 'x' and leaves the rest unchanged. Currently, it is mapped over the entire ServerState list, but it would be easy to map over only the group to which the player belongs. Here is how the sub-list is isolated:

```haskell
subState :: Text -> Text -> [(Text,Int,Text,WS.Connection)] -> [(Text,Int,Text,WS.Connection)]
subState name gr state  | gr /= solo  = [ (a,b,c,d) | (a,b,c,d) <- state, gr == c ]
                        | gr == solo = [ (a,b,c,d) | (a,b,c,d) <- state, name == a]

```
'solo' is the group name for players who have not joined a group. solo players don't get information about other solo players, so name is used to narrow down the result to only one item in the ServerState list. Group members who look at computer generated solutions are temporarily assigned to group 'solo'. When they close the solutions interface they rejoin their old group, keeping the score they had when they left to look at solution.

The information arriving from the server is a comma-separated entity treated as a list of Char, a/k/a String. It gets separated at the commas to a list of type [String] and the item corresponding to the user's Group is converted to Text. It never had type Group, although the list comprehension treats it as the Group element in a ServerState tuple. Haskell's type safety gets diluted through its continual exchange of information with browsers.

I'll have a little more to say about isolating groups in ServerState later. There was a time during development when the browsers had to screen messages. When I turned my attention to having the server send no more messages than necessary, it was pleasant to be coding Haskell again where list comprehensions and pattern matching tell the compiler what is wanted in the simplest possible terms, and the compiler complies with exactly what was requested.

##The timer

The timer can be started either by clicking SCORE! or by clicking IMPOSSIBLE. SCORE! starts a ten-second coundown during which the first computation must be completed or a point will be forfeited. Completing the first computation before ten seconds have expired starts another ten-second countdown. If the result of the second computation is the number '20', the player is awarded a point; otherwise, another ten-second countdown begins. If the time expires or the computation doen not result in '20', a point is deducted. Otherwise the result is '20' and the player gains a point.

Clicking Score starts a sixty-second countdown. If the time expires, the player gains one point. If, during the countdown, a player clicks SCORE! and computes the number '20', that player gains a point and the player who clicked IMPOSSIBLE loses two points. 

The SCORE! button is in B2's render function. it is defined as follows:

```js
          <button onClick={this.handleScore2}
          style={this.style7(cr19,cr190,scoreDisplay2)}
          onMouseEnter={() => {this.mouse[19] = 'blue'; this.mouse[190]  = '#01afaf' }}
          onMouseLeave={() => {this.mouse[19] = '#000'; this.mouse[190]  = 'darkred' }} >
            SCORE!
          </button>

```
style7() handles style for several buttons. The first two parameters are 'backgroundColor' and 'borderColor'. The third parameter matches 'display' in style7(). It's values are either 'none' or 'inline'. Player actions make it come and go. Clicking IMPOSSIBLE removes it and substiturs a button that looks just like it for use in iterrupting the 60-second countdown. 
Here is the click handler:

```js
  handleScore () {
    let name = this.state.name;
    let group = this.state.group;
    DES_ws.send( `CY#$42,${group},${name},${name}` );
  }
```
The message "`CY#$42,${group},${name},${name}`" arrives at the server and is broadcast to all members of the sender's group. That is all the server does with it. It determines which sockets correspond to the members of the sender's group and sends the message to each of them. Here is what the browsers do with the message:
```
  case "CY#$42": // Triggered by clicking "SCORE!".
    that.setState( {
      scoreClicker: extra,
      score: true,
      message: '',
      DS_T: 10,
      impossibleDisplay: 'none',
      solutionsDisplay: 'none',
      timerDisplay: 'inline',
      scoreDisplay: 'none',
      rollDisplay: 'none'
    } )
    if (extra !== name) {      
        that.setState({numDisplay: 'none'}) // Players can see calculations after wait.
        setTimeout ( function() {
          that.setState({solutionsDisplay: 'inline'});
        },8000 )
    }
  break;
```
If the countdown reaches 0, a message is circulated to all members of the group (through the server) setting the value of DS\_T in the following manner:

```js
this.setState({DS\_T: `10 seconds expired. Deduct one point from ${scoreClicker}`});
this.setState({DS\_T: `60 seconds expired. One point for ${impossibleClicker}`});
this.setState( {DS_T: `10 seconds expired. One point awarded to ${that.state.impossibleClicker}
	One point deducted from ${this.state.interruptClicker}`} 
```
This stops the clock by setting DS\_T to a string value. What is sent to the server, and the resulting messages broadcast to other group member, depends on whether the browser belongs to a player who clicked SCORE! to begin a round, a player who clicked IMPOSSIBLE, or a player who clicked SCORE! after IMPOSSIBLE was clicked. In any given round there will be only one player who clicked SCORE! since initially clicking SCORE! causes the IMPOSSIBLE button to disappear preventing the second SCORE! button from appearing. Clicking IMPOSSIBLE causes the first SCORE! button to disappear. Every player in the group has a timer, but only the ones with 'name == scoreClicker' or 'name == impossibleClicker' send messages updating the score board. Clicking SCORE! or IMPOSSIBLE sets the scoreClicker and impossibleClicker variables to the clicker's login name.

As you see below, DS\_T > 0 causes the countdown to continue while DS\_T == 0 causes a message, or messages, to go out to the server.

```js
  setInterval( () => {
    let name = this.state.name;
    let group = this.state.group;
    let scoreClicker = this.state.scoreClicker;
		let impossibleClicker = this.state.impossibleClicker;
		let interruptClicker = this.state.interruptClicker
		let score = this.state.score;
		let impossible = this.state.impossible;
		let interrupt = this.state.interrupt;

    if ( this.state.DS_T > 0 ) {
      this.setState({
        DS_T: this.state.DS_T - 1,
        timeSize: 40
      });
      this.setState({info: this.state.DS_T});

    }

    if ( this.state.DS_T*1 === 0 ) {
      this.setState ({
            message1: 0,  // Wipes the old numbers.
            message2: 0,
            message3: 0,
            message4: 0,
            info: '',
            timeSize: 20, // Returns number display to normal size.
            rollDisplay: 'inline', // Displays the ROLL button.
            solutionsDisplay: 'inline',
            DS_t: -1
        })
  	  let z = scoreClicker === name;
  		let z2 = impossibleClicker === name;
  		let z3 = interruptClicker === name;
  	  let gr = group;
      if (!interrupt) {
				if (z) {
        	DES_ws.send(`CG#$42,${gr},${name},-1`);
          DES_ws.send(`CH#$42,${gr},${name},10 seconds expired. Deduct one point from ${scoreClicker}`);

      	}
			  else if (z2) {
        	DES_ws.send(`CG#$42,${gr},${name},1`);
          DES_ws.send(`CH#$42,${gr},${name},60 seconds expired. One point for ${impossibleClicker}`);

				}
			}
      else if (z3) {
        DES_ws.send(`CG#$42,${gr},${interruptClicker},-1`);
				DES_ws.send(`CH#$42,${group},${interruptClicker},10 seconds expired. One point awarded to ${that.state.impossibleClicker}. One point deducted from ${interruptClicker}.`);

				DES_ws.send(`CG#$42,${gr},${impossibleClicker},1`);
		  }
		}
  }, 1000 )
}

```
That is the gist of how the timer works. Because DS\_t is a plain Javascript variable, it has no set type. That makes it convenient to stop the integer countdown by assigning a string value to DS\_t. The counter can't use a string, but it doesn't complain. It just stops counting. 

I won't present any more details here. I think they can be deduced by following the code and besides, most readers are interested in a general overview of how one person (me) used a Haskell Websockets server to create a multiplayer dice game with user-defined competition groups and chat rooms. 

##More on How the server determines which players are sent broadcast messages.

As previously mentioned, the server's broadcast function doesn't broadcast to every Client in the ServerState list. Instead, it broadcasts to each member of the group to which the sender of a triggering message belongs. So, for example, if the server receives the Message Board message 'CD#$42,GroupA,Fred,Fred: Hello out there.', it parses it just as it parses all incoming messages:
```
msg <- WS.receiveData conn
let msgArray = splitOn "," (T.unpack msg)
let group = T.pack (msgArray !! 1)
let sender = T.pack (msgArray !! 2)
let extra = T.pack (msgArray !! 3)
let extraNum = read (msgArray !! 3) :: Int
let range = get4 msgArray  -- 7 items in msgArray
```
It then runs:
```
do
    st <- atomically $ readTMVar state
    let subSt = subState group st
    broadcast msg subSt
```
Here is the function 'subState' again:
```
subState :: Text -> Text -> [(Text,Int,Text,WS.Connection)] -> [(Text,Int,Text,WS.Connection)]
subState name gr state  | gr /= solo  = [ (a,b,c,d) | (a,b,c,d) <- state, gr == c ]
                        | gr == solo = [ (a,b,c,d) | (a,b,c,d) <- state, name == a]
```
That might not be the most efficient way to get the list of Client tupples belonging to the sender's group, but I think it is wonderfull that, in Haskell, we can simply say what we want and get it without further ado. The function matches every Client in the designated group unless the group is 'solo'. In that case, the unique player with the designated name is the sole member of the substate list.
```
A player wanting to change groups sends a string prefixed by 'CO#$42', The server deals with it as follows:
```
    else if "CO#$42" `T.isPrefixOf` msg
        then
            mask_ $ do
                old <- atomically $ takeTMVar state
                let new = changeGroup sender extra old
                atomically $ putTMVar state new
                let subState1 = subState sender group new
                let subState2 = subState sender extra new
                let x = "CB#$42," `mappend` group `mappend` "," `mappend` sender `mappend` "," `mappend` T.concat (intersperse "<br>" (textState subState1))
                let y = "CB#$42," `mappend` extra `mappend` "," `mappend` sender `mappend` "," `mappend` T.concat (intersperse "<br>" (textState subState2))
                broadcast y subState2
                if group /= "solo"
                   then
                   broadcast x subState1
                   else
                   return ()
```
'changeGroup()' takes three arguments: 'sender', 'extra', and 'old'. 'extra' is the fourth element of the list produced by parsing the comma-separated string that arrives from the browser. In this case, it is the name of a group. old is the ServerState list before the group membership change. Here is 'changeGroup()':

```
changeGroup :: Text -> Text -> ServerState -> ServerState
changeGroup name group = map (newGroup name group
```

As you see, it maps 'newGroup' over the ServerState list. Here is newGroup:

```
newGroup :: Text -> Text -> Client -> Client
newGroup name group (a, b, c, d)   | name == a  = (a, 0, group, d)
                                   | otherwise = (a, b, c, d)

```

Notice that the new group member starts with a score of '0'. The exception to that rule occurs when a player re-joins a group after looking at computer-generated solutions. The code in such cases is:

```

newGroupKeepScore :: Text -> Text -> Client -> Client
newGroupKeepScore name group (a, b, c, d)   | name == a  = (a, b, group, d)
                                   | otherwise = (a, b, c, d)

changeGroupKeepScore :: Text -> Text -> ServerState -> ServerState
changeGroupKeepScore name group = map (newGroupKeepScore name group)
```

The second member of the tupple is 'b', the old score, rather than always being '0'. 'group' in this function is 'solo' when a player displays soloutions and the old group, which is saved in the browser, when the solutions are hidden. After hiding the solutions, a player must wait until the next round to resume competition.

Finally, here is the broadcast function, just as it first appeared in the Haskell Websockets library:
```
broadcast :: Text -> ServerState -> IO ()
broadcast message clients = do
    T.putStrLn message
    forM_ clients $ \(_ , _, _, conn) -> WS.sendTextData conn message

```
##The Front End



## APPENDIX
### IMPOSSIBLES
The essence of the Score calculation algorythm in the module Fm is contained in the "impossibles.hs" file. Fm has much formatting code, which is a distraction when evaluating the algorythm.

impossibles.hs computes all dice combinations which cannot be made into the number "20" in two or three stages, as required by the game. In 1.5 seconds, it finds all 104 such combinations using a list comprehension on the seven list comprehensions which cover all possible computations. The five operations are defined as follows:

```haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.List
import System.CPUTime

notWhole :: Double -> Bool
notWhole x = fromIntegral (round x) /= x

cat :: Double -> Double -> Double
cat l m  | (whole l) && (whole m) && m >= 0 && l /= 0  = read ((show $ fRound l) ++ (show $ fRound m))
         | otherwise = 8.888

f :: Double -> String
f x = show (round x)

ops =  [cat, (+), (-), (*), (/)]
```

The seven algorithms necessary to perform every possible computation are:

```haskell
calc :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op2 (op1 a' b') c' == 20]

calc2 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc2 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op2 a' (op1 b' c') == 20]

calc3 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc3 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op1 a' b') (op2 c' d') == 20]

calc4 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc4 a b c d = [ (a',b',c',d')  |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op2 (op1 a' b') c') d' == 20]

calc5 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op2 a' (op1 b' c')) d' == 20]

calc6 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 a' (op2 (op1 b' c') d') == 20]

calc7 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 a' (op2 b' (op1 c' d')) == 20]
```

It is easy to see that there are seven ways to order two or three sequential computations on four numbers. They can be represented by

```javascript
a bc
ab c
ab cd
(a bc)d
(ab c)d
a(b cd)
a(bc d)
```

Those are the combinations used in the seven calc functions. The list comprehension works on all permutations of the order of the four numbers in receives, so c ba is covered by a bc. Next, I wanted to find out if all seven algorithms are necessary to find at least one solution, so I wrote this:

```haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.List
import System.CPUTime

notWhole :: Double -> Bool
notWhole x = fromIntegral (round x) /= x

cat :: Double -> Double -> Double
cat l m  | (whole l) && (whole m) && m >= 0 && l /= 0  = read ((show $ fRound l) ++ (show $ fRound m))
         | otherwise = 8.888

f :: Double -> String
f x = show (round x)

ops =  [cat, (+), (-), (*), (/)]

calc :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op2 (op1 a' b') c' == 20]

calc2 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc2 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op2 a' (op1 b' c') == 20]

calc3 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc3 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op1 a' b') (op2 c' d') == 20]

calc4 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc4 a b c d = [ (a',b',c',d')  |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op2 (op1 a' b') c') d' == 20]

calc5 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op2 a' (op1 b' c')) d' == 20]

calc6 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 a' (op2 (op1 b' c') d') == 20]

calc7 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 a' (op2 b' (op1 c' d')) == 20]



only_calc = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     not (null $ calc a b c d), null $ calc2 a b c d, null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d, null $ calc6 a b c d,
                     null $ calc7 a b c d ]


only_calc2 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, not (null $ calc2 a b c d), null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d, null $ calc6 a b c d,
                     null $ calc7 a b c d ]

only_calc3 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, null $ calc2 a b c d, not (null $ calc3 a b c d),
                     null $ calc4 a b c d, null $ calc5 a b c d, null $ calc6 a b c d,
                     null $ calc7 a b c d ]

only_calc4 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, null $ calc2 a b c d, null $ calc3 a b c d,
                     not (null $ calc4 a b c d), null $ calc5 a b c d, null $ calc6 a b c d,
                     null $ calc7 a b c d ]

only_calc5 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, null $ calc2 a b c d, null $ calc3 a b c d,
                     null $ calc4 a b c d, not (null $ calc5 a b c d), null $ calc6 a b c d,
                     null $ calc7 a b c d ]

only_calc6 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, null $ calc2 a b c d, null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d, not (null $ calc6 a b c d),
                     null $ calc7 a b c d ]

only_calc7 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, null $ calc2 a b c d, null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d, null $ calc6 a b c d,
                     not (null $ calc7 a b c d )]

main = do
    print "*****************************___only_calc"
    t1 <- getCPUTime
    mapM_ print only_calc
    print " "

    print "*****************************___only_calc2"
    mapM_ print only_calc2
    print " "

    print "*****************************___only_calc3"
    mapM_ print only_calc3
    print " "

    print "*****************************___only_calc4"
    mapM_ print only_calc4
    print " "

    print "*****************************___only_calc5"
    mapM_ print only_calc5
    print " "

    print "*****************************___only_calc6"
    mapM_ print only_calc6
    print " "

    print "*****************************___only_calc7"
    mapM_ print only_calc7
    t2 <- getCPUTime
    let t = fromIntegral (t2-t1) * 1e-12
    print t
    print " "
```

Here is what I got:

```javascript
e@e:~/b0$ ./analysis_A
"*****************************___only_calc"
[1.0,3.0,11.0,15.0]
[1.0,3.0,11.0,19.0]
[1.0,6.0,9.0,20.0]
[1.0,6.0,10.0,20.0]
[1.0,6.0,11.0,20.0]
[2.0,2.0,11.0,15.0]
[2.0,2.0,11.0,17.0]
[3.0,3.0,3.0,13.0]
[3.0,3.0,7.0,17.0]
[3.0,4.0,9.0,16.0]
[3.0,4.0,11.0,14.0]
[4.0,4.0,6.0,17.0]
[5.0,5.0,5.0,11.0]
[5.0,5.0,5.0,13.0]
[5.0,5.0,5.0,17.0]
" "
"*****************************___only_calc2"
" "
"*****************************___only_calc3"
[1.0,1.0,1.0,11.0]
[1.0,1.0,7.0,17.0]
[1.0,1.0,12.0,12.0]
[1.0,6.0,6.0,6.0]
[1.0,6.0,9.0,9.0]
[3.0,3.0,6.0,6.0]
[3.0,4.0,7.0,18.0]
[3.0,6.0,7.0,14.0]
[5.0,5.0,6.0,17.0]
[5.0,6.0,6.0,6.0]
" "
"*****************************___only_calc4"
[1.0,1.0,4.0,11.0]
[1.0,4.0,9.0,9.0]
[1.0,4.0,9.0,19.0]
[1.0,5.0,11.0,11.0]
[1.0,6.0,6.0,12.0]
[1.0,6.0,11.0,11.0]
[3.0,6.0,9.0,12.0]
[6.0,6.0,7.0,18.0]
[6.0,6.0,9.0,14.0]
" "
"*****************************___only_calc5"
[1.0,3.0,8.0,20.0]
[3.0,3.0,10.0,17.0]
[3.0,4.0,10.0,16.0]
" "
"*****************************___only_calc6"
" "
"*****************************___only_calc7"
8.27113
" "
```

This shows that there is no solution that only calc2, only calc6, or only calc7 can find. Next, I checked all combinations of these three:

```javascript
{-# LANGUAGE OverloadedStrings #-}

import Data.List
import System.CPUTime

notWhole :: Double -> Bool
notWhole x = fromIntegral (round x) /= x

cat :: Double -> Double -> Double
cat l m  | (whole l) && (whole m) && m >= 0 && l /= 0  = read ((show $ fRound l) ++ (show $ fRound m))
         | otherwise = 8.888

f :: Double -> String
f x = show (round x)

ops =  [cat, (+), (-), (*), (/)]

calc :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op2 (op1 a' b') c' == 20]

calc2 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc2 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op2 a' (op1 b' c') == 20]

calc3 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc3 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op1 a' b') (op2 c' d') == 20]

calc4 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc4 a b c d = [ (a',b',c',d')  |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op2 (op1 a' b') c') d' == 20]

calc5 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op2 a' (op1 b' c')) d' == 20]

calc6 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 a' (op2 (op1 b' c') d') == 20]

calc7 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 a' (op2 b' (op1 c' d')) == 20]


only_calc2_or_6 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, not (null $ calc2 a b c d), null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d, not (null $ calc6 a b c d),
                     null $ calc7 a b c d ]

only_calc2_or_7 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, not (null $ calc2 a b c d), null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d, null $ calc6 a b c d,
                     not (null $ calc7 a b c d) ]

only_calc6_or_7 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, null $ calc2 a b c d, null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d, not (null $ calc6 a b c d),
                     not (null $ calc7 a b c d )]


only_calc2_or_6_or_7 = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, not (null $ calc2 a b c d), null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d, not (null $ calc6 a b c d),
                     not (null $ calc7 a b c d )]

main = do
    print "*****************************___only_calc2_or_6"
    t1 <- getCPUTime
    mapM_ print only_calc2_or_6
    print " "

    print "*****************************___only_calc2_or_7"
    mapM_ print only_calc2_or_7
    print " "

    print "*****************************___only_calc6_or_7"
    mapM_ print only_calc6_or_7
    print " "

    print "*****************************___only_calc2_or_6_or_7"
    mapM_ print only_calc2_or_6_or_7
    t2 <- getCPUTime
    let t = fromIntegral (t2-t1) * 1e-12
    print t
```

And here is what I got:

```javascript
    e@e:~/b0$ ./analysis_B
"*****************************___only_calc2_or_6"
" "
"*****************************___only_calc2_or_7"
" "
"*****************************___only_calc6_or_7"
" "
"*****************************___only_calc2_or_6_or_7"
[2.0,5.0,12.0,12.0]
3.385727
```

There are no rolls of the dice that can be found only by some pair of these functions, and (2,5,12,12) is the only roll that can be found by all three, but none of the other algorithms (calc, calc3, calc4, and calc5). Those four along with any one of calc2, calc6, or calc7, are sufficient to find at least one solution if a roll is solvable. A corollary is that if calc, calc2, calc3, calc4, and calc5 can't find a solution, calc6 and calc7 won't either. I tested this by removing calc6 and calc7 from impossibles.hs and renaming it impossibles2.hs. Like impossibles.hs, it found the 104 impossible rolls, only in 1.33 instead of 1.50 seconds. The module Fm uses the seven algorithms to find solutions to random rolls or numbers entered by Score players. It massages the output into a single line of Text with solutions separated by "br" in <> brackets. The browsers receive the Text as a Javascript string which, when appended to a div, displays the solutions neatly in a column.

### All 104 Impossible Rolls
Here is the code for impossibles2.hs:

```haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.List
import System.CPUTime

notWhole :: Double -> Bool
notWhole x = fromIntegral (round x) /= x

cat :: Double -> Double -> Double
cat l m  | (whole l) && (whole m) && m >= 0 && l /= 0  = read ((show $ fRound l) ++ (show $ fRound m))
         | otherwise = 8.888

f :: Double -> String
f x = show (round x)

ops :: [Double -> Double -> Double]
ops =  [cat, (+), (-), (*), (/)]

calc :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op2 (op1 a' b') c' == 20]

calc2 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc2 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op2 a' (op1 b' c') == 20]

calc3 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc3 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op1 a' b') (op2 c' d') == 20]

calc4 :: Double -> Double -> Double -> Double -> [(Double, Double, Double, Double)]
calc4 a b c d = [ (a',b',c',d')  |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op2 (op1 a' b') c') d' == 20]

calc5 a b c d = [ (a',b',c',d') |
                        [a',b',c',d'] <- nub(permutations [a,b,c,d]),
                            op1 <- ops,
                            op2 <- ops,
                            op3 <- ops,
                            op3 (op2 a' (op1 b' c')) d' == 20]

impossibles = [ [a, b, c, d] | a <- [1..6], b <- [1..6], c <- [1..12], d <- [1..20],
                     a <= b, b <= c, c <= d,
                     null $ calc a b c d, null $ calc2 a b c d, null $ calc3 a b c d,
                     null $ calc4 a b c d, null $ calc5 a b c d ]

main = do
    t1 <- getCPUTime
    mapM_ print impossibles
    t2 <- getCPUTime
    let t = fromIntegral (t2-t1) * 1e-12
    print t
```


