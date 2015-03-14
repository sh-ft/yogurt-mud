All examples are also available in the download from hackage.

## Minimal example ##

After installing Yogurt, create a file called Minimal.hs and put the following text in it:

```
module Minimal where

import Network.Yogurt

newmoon :: Session
newmoon = session
  { hostName   = "eclipse.cs.pdx.edu"
  , portNumber = 7680
  }
```

This is the smallest example: a single session specifying the host and port to connect to, but no scripting.

Then start Yogurt by issuing:

```
$ yogurt Minimal
```

## A more involved example ##

The following example demonstrates some of the features Yogurt boasts.

```
module NewMoon where

import Network.Yogurt
import System.Cmd
import Data.Char

newmoon :: Session
newmoon = session
  { hostName = "eclipse.cs.pdx.edu"
  , portNumber = 7680
  , mudProgram = \reload -> do
      -- The reload argument allows reloading this Haskell file during a session.
      -- Let's make a command for it so we can call it while playing.
      mkCommand "reload" reload
      installHooks      
  }

installHooks :: Mud ()
installHooks = do
  -- Logs are stored in a timestamped file in the local directory.
  startLogging "NewMoon"

  -- Log in automatically.
  mkTriggerOnce "^Enter your name:" $ do
    sendln "username"
    sendln "password"

  -- Sound system bell whenever you receive a tell.
  mkTrigger "^[^ ]+ tells you: " bell

  -- Send a carriage return every 5 minutes to keep the connection alive.
  -- Most MUDs don't like this...
  mkTimer (5 * 60 * 1000) (sendln "")

  -- Enable calling system commands during a session.
  mkCommand "system" $ do
    group 1 >>= liftIO . system
    return ()

  -- | Split commands in two on semicolons.
  mkPrioHook 100 Remote ";" $ do
    before >>= matchMoreOn  . (++ "\n")
    after  >>= matchMoreOn'

  -- Enable speedwalks, e.g. "5w" expands to "w;w;w;w;w"
  mkHook Remote "^[0-9]+[neswud]$" $ do
    (n, dir) <- fmap (span isDigit) (group 0)
    sequence $ replicate (read n) (sendln dir)
  
  -- A very simple alias: make "fb" expand to "cast fireball"
  mkAlias "fb" "cast fireball"
  
  -- Create a command to teleport to a specific place.
  mkCommand "tp" $ do
    -- Find out where to go.
    dest <- group 1
    if all isSpace dest
      then do
        echoln "Teleport to where?"
      else do
        sendln "cast teleport"
  
        -- Wait for the spell to get ready...
        mkTriggerOnce "^You feel ready to teleport now" $ do
          sendln ("teleport to" ++ dest)
        return ()
  
  -- Show all currently install hooks whenever "hooks" is entered.
  mkCommand "hooks" (allHooks >>= echo . unlines . map show)

  return ()
```