---
title: Wykład 15
---

# Wykład 15

> *(na kartce)*

## Powrót do skutków ubocznych

    #!haskell
    data Request = ReadReq | WriteReq Char
    data Response = ReadResp Char | WriteResp
    type Dialog = [Response] -> [Request]

    upcase :: Char -> Char
    main :: Dialog

    main = \resp ->
        ReadReq :
            case resp of
                ReadResp c:resp' -> 
                    (WriteReq $ upcase c) :
                        case resp' of
                            WriteResp : resp'' -> main resp

Podobno kiedyś nawet tak programowano w Haskellu! Ale da się lepiej - abstrakcyjnie, używając monad.

Zamiast operować na konkretnych strumieniach, możemy stworzyć typ `IO`, który by to ogarnął lepiej.

    #!haskell
    main :: IO ()
    return :: a -> IO a
    (>>=) :: IO a -> (a -> IO b) -> IO b

Jak na razie to jest zwykła monada, rozszerzymy to o operacje specyficzne dla IO.

    #!haskell
    getChar :: IO Char
    putChar :: Char -> IO ()

Mając takie coś, możemy bardzo schludnie napisać `main`:

    #!haskell
    main = do
        c <- getChar
        putChar $ upcase c
        main

## Coś o monadach stanowych

    #!haskell
    newtype StateM s a = StateM (s -> (a,s))

    class Monad (m s) => StateMonad (m s) where
        update :: (s -> s) -> m s ()

    instance Monad (StateM s) where
        -- ...

    instance StateMonad (StateM s) where
        -- ...

Chcąc mieć parser, musimy umieć przerobić jedną monadę na drugą.

    #!haskell
    class MonadTransformer (t:: (*->*) -> (*->*)) where
        lift :: Monad m => m a -> t m a

    newtype StateT (s::*) (m :: * -> *) (a :: *) = StateT (s -> m (a,s))
    instance Monad m => Monad (StateT s m) where
        return a = StateT (\s -> return (a,s))

        (StateT g) >>= f = StateT (\s -> g s >>= (\(a,s') -> let StateT h = f a in h s'))

    instance MonadTransformer (StateT s) where
        lift x = StateT (\s -> x >>= (\a -> return (a,s)))

    instance Monad m => StateMonad (StateT s m) where
        update f = StateT (\s -> return ((), f s))

Teraz możemy przenosić operacje z monad do innych:

    #!haskell
    tail :: [a] -> [a]
    lift . tail :: [a] -> StateT s [] a

Pozwala nam to np. na dodanie stanu do dowolnej monady.