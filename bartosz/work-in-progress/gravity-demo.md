# Gravity Demo

This is not a tutorial. I just put this code together to show the power of our snippet running system. This project is also maintained on [GitHub](https://github.com/BartoszMilewski/gravity-sim).


It's an example of a client server application capable of performing calculation on the server and displaying graphics on the client. Press the run button and scroll to the bottom to see a simulation. You can also click on the button there to view the simulation in a separate window. 

What you will see is the movement of the inner planets in our Solar System according to Newton's laws of gravity. I also threw in some comets, just for fun.

``` active haskell web

{-# START_FILE World.hs #-}
-- show

{-# LANGUAGE ScopedTypeVariables #-}

module World (
  -- read a world from a file
  readWorld,

  solarWorld,
  -- a 4-body world
  world4 
) where

import Control.Exception  (catch)
import System.Exit        (exitFailure)
import Types

width, height :: Int    -- extent of the window; origin is in the center
width  = 600
height = 600

-- Read a world model from the given file
--
readWorld :: FilePath -> IO World
readWorld fname
  = do
      contents <- readFile fname
      readIO contents
   `catch` \(exc::IOError) ->
     do 
       putStrLn $ "Fatal error: can't read world description\n" ++ show exc
       exitFailure

solarWorld :: World
solarWorld = World 0 distanceScale (earthMass / 10000) 750
                      [ Particle (Mass sunMass) 
                                 (Pos 0 0) (Vel 0 0)
                      , Particle (Mass cometMass) 
                                 (Pos cometDist 0) (Vel 0 cometVelocity)
                      , Particle (Mass cometMass) 
                                 (Pos (-cometDist) (-cometDist)) (Vel 5000 (-5000))
                      , Particle (Mass cometMass) 
                                 (Pos 2.0e11 1.0e11) (Vel (-2500) 5000)
                      , Particle (Mass earthMass) 
                                 (Pos earthDist  0) (Vel 0 earthVelocity)
                      , Particle (Mass venusMass) 
                                 (Pos venusDist  0) (Vel 0 venusVelocity)
                      , Particle (Mass mercuryMass) 
                                 (Pos mercuryDist  0) (Vel 0 mercuryVelocity)]
  where
    sunMass         = 1.9891e30
    earthDist       = 152098232e3   -- Aphelion
    earthMass       = 5.9736e24
    earthVelocity   = 29.78e3
    venusDist       = 1.08e11
    venusMass       = 4.869e24
    venusVelocity   = 35e3
    mercuryDist     = 4.6e10
    mercuryMass     = 3.3e23
    mercuryVelocity = 49.88e3
    cometDist       = 2.0e11
    cometMass       = 1.0e20
    cometVelocity   = 7000
    --
    distanceScale = (fromIntegral height * 0.4) / earthDist 

world4 :: World
world4 = World 0 0.5 9.42590890872e11 1
               [ Particle (Mass 1e16) (Pos (-100) 30) (Vel 0 (-65))
               , Particle (Mass 1e16) (Pos 240 0)     (Vel (-40) 30)
               , Particle (Mass 1e16) (Pos 50 200)    (Vel 0 (-30))
               , Particle (Mass 1e15) (Pos 0 (-300))  (Vel 0 5)]


{-# START_FILE Physics.hs #-}
-- show

module Physics (force) where

import Types

-- For floating point comparisons
epsilon :: Float
epsilon = 0.001

-- Gravitational constant
bigG :: Float
bigG = 6.67428e-11        -- in m^3 kg^(-1) s^(-2)

-- Given two particles, determine the acceleration exerted by the second on the first.
--
-- As a special case, the force is zero if both particles are closer than 
-- a minimal epsilon distance.
--
force :: Particle -> Particle -> Accel
force (Particle (Mass _) (Pos x1 y1) _) (Particle (Mass m2) (Pos x2 y2) _)
  | d < epsilon = Acc 0 0
  | otherwise   = Acc (absAccel * dx / d) (absAccel * dy / d) 
  where
    dx       = x2 - x1
    dy       = y2 - y1
    dsqr     = dx * dx + dy * dy
    d        = sqrt dsqr
    absAccel = bigG * m2 / dsqr

{-# START_FILE Simulation.hs #-}
-- show

module Simulation (moveParticle, accelerate, advanceWorld) where

import Types
import Physics
import Control.Parallel.Strategies

-- Move a particle according to its velocity for the given 
-- number of (simulated) seconds.
--
moveParticle :: Float -> Particle -> Particle
moveParticle dt (Particle m (Pos x y) (Vel vx vy)) =
  Particle m (Pos (x + dt * vx) (y + dt * vy)) (Vel vx vy)
    
-- Accelerate a particle in dependence on the gravitational force 
-- exerted by all other particles for
-- the given number of (simulated) seconds.
-- force :: Particle -> Particle -> Accel
accelerate :: Float -> [Particle] -> [Particle]
accelerate dt particles =
    parMap rseq acc particles
  where
    acc particle =
      foldl addAcc particle particles
    addAcc myParticle@(Particle m pos (Vel vx vy)) otherParticle =
      let (Acc ax ay) = force myParticle otherParticle
      in
        Particle m pos (Vel (vx + dt * ax) (vy + dt * ay))

-- Progressing the world state
--
advanceWorld :: Float -> World -> World
advanceWorld dtReal world =
  let dt = dtReal * usrToWrldTime world
      newParticles = map (moveParticle dt) (accelerate dt $ parts world)
  in
      world { parts = newParticles }

{-# START_FILE Main.hs #-}
-- show

{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
module Main where

import Yesod
import Types
import System.Environment (getEnv)
import qualified Control.Exception as E

import World
import Simulation

-- to do:
-- add Pause button
-- set maximum time for simulation
-- make curWorld a local variable

data Gravity = Gravity

instance Yesod Gravity

mkYesod "Gravity" [parseRoutes|
  /        HomeR    GET
  /advance AdvanceR POST
  /solar   SolarR   GET
  /world4  World4R  GET
|]

boxColor, bodyColor :: String

boxColor   = "#00"
bodyColor  = "#333"

boxSizeX, boxSizeY  :: Int
boxSizeX    = 600
boxSizeY    = 600

framesPerS :: Int
framesPerS = 16

getHomeR :: HandlerT Gravity IO Html
getHomeR = defaultLayout $ do
  setTitle "Gravity"

  [whamlet|
    <div #box>
      <p>
      <canvas #sky width=#{boxSizeX} height=#{boxSizeY}> 
         Your browser doesn't support HTML 5
      <p>
        Gravitational interaction demo based on one of 
        <a href="http://www.cse.unsw.edu.au/~chak/" target="_blank">Manuel Chakravarty</a>'s 
        Haskell course exercises. The simulation is done in Haskell on the server. 
        Client code uses HTML 5 to display instantaneous positions of bodies. 
        It communicates with the (stateless) server using JSON. The web site is written in 
        <a href="http://www.yesodweb.com/" target="_blank">Yesod</a>.
        <div>
          <button #reset>Reset
          <select>
            <option value="solar"> Inner planets
            <option value="world4"> Four stars
  |]

  toWidget [cassius|
    #box
      width:#{show boxSizeX}px
      height:#{show boxSizeY}px
      margin-left:auto
      margin-right:auto
    canvas
      background-color:#{boxColor}
    body
      background-color:#{bodyColor}
      color:#eee
      font-family:Arial,Helvetica,sans-serif
      font-size:small
    a
      text-decoration:none
      color:#bdf
    #sky
      border:1px solid #888
  |]

  addScriptRemote "//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"

  toWidget [julius|
    var urls = { // map from simulation names to urls
      solar:   "@{SolarR}", 
      world4:  "@{World4R}" 
    };
    // Important: changes to global variables
    // may only happen inside JSON handlers
    var interval = 1000.0 / #{toJSON framesPerS};
    var curSimName = null;
    var curWorld = null; // constantly changing world
    var skip = 0; // skip n frames if congested

    $(document).ready(function() {
        $("#reset").click(function() {reset(curSimName);});
        $("select").change(function() {
            var str = $("select option:selected").val();
            reset(str);
        });
        $("select option[value='solar']").attr('selected', 'selected');
        reset("solar");
        setInterval(advance, interval);
    });

    function reset(simName) {
        if (!urls[simName]) 
            alert("Error: invalid simulation: " + simName);
        $.getJSON(urls[simName], function(newWorld){
            newWorld.seqNum = curWorld? curWorld.seqNum + 1: 0;
            curSimName = simName;
            curWorld = newWorld;
        });
    }

    // Called in a loop
    function advance() {
        if (skip == 0) {
            drawWorld();
            refreshWorld();
        } else
            skip -= 1;
    }

    function refreshWorld(simName) {
        // Get new world from server
        $.ajax(
        {
           "data"    : JSON.stringify(curWorld),
           "type"    : "POST",
           "url"     : "@{AdvanceR}",
           "success" : updateWorld
        });
    }

    // Handler called with new world
    function updateWorld(newWorld)
    {
       if(!curWorld) alert("null world!");
       var lag = curWorld.seqNum - newWorld.seqNum;
       if (lag == 0) {
           curWorld = newWorld;
           curWorld.seqNum += 1;
       } else if (lag > 0)
           skip = lag;
       else
           alert("Time travel discovered!")
    }

    var dimX = #{toJSON boxSizeX};
    var dimY = #{toJSON boxSizeY};

    function drawWorld() {
        if (!curWorld) return true; // might happen

        var canvas = document.getElementById('sky');
        var ctx = canvas.getContext('2d');
        ctx.clearRect(0, 0, dimX, dimY);
        ctx.fillStyle = "white";

        // Draw particles
        var partsInView = 0;
        for (var j = 0; j < curWorld.parts.length; j++) {
          var part = curWorld.parts[j];
          var size = Math.log(part.pmass/curWorld.pixInKg) / Math.LN10;
          if (size < 2) size = 2;
          var x = dimX/2 + curWorld.pixInM * part.ppos.posx;
          var y = dimY/2 + curWorld.pixInM * part.ppos.posy;
          if ( x > -10 && x < dimX + 10 && y > -10 && y < dimY + 10) {
              partsInView += 1;
              ctx.beginPath(); 
              ctx.arc(x, y, size/2, 0, Math.PI * 2, true); 
              ctx.fill();
          }
      }
      return partsInView != 0;
    }
  |]

--------------------
-- Server side logic
--------------------

postAdvanceR :: Handler Value
postAdvanceR = do
    -- Parse the request body to a data type as a JSON value
    world <- requireJsonBody
    -- user time in seconds
    let userTime = 1.0 / fromIntegral framesPerS
    let worldTime = userTime * usrToWrldTime world
    -- do the simulation
    returnJson $ advanceWorld worldTime world

getSolarR  :: Handler Value
getSolarR  = returnJson solarWorld

getWorld4R :: Handler Value
getWorld4R = returnJson world4

main :: IO ()
main = do
    portEither <- getPortEither
    let port = case portEither of
                        Right val -> read val
                        Left _    -> 3000
    -- start the server
    warp port Gravity
  where
    -- try to get the port from environment
    getPortEither :: IO (Either IOError String)
    getPortEither = E.try (getEnv "PORT")

```