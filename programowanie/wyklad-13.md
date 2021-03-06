---
title: Monady wszędzie
---

# Monady wszędzie

## Polimorfizmy

Polimorfizm w zasadzie opiera się na nazywaniu kilku różnych rzeczy tak samo.
Polimorfizmem jest np. zrobienie operacji `+` do różnych działań. Nazywa się to wtedy **przeciążaniem**. Takimi operacjami mogłyby być dodawanie Integerów i dodawanie Floatów. Od strony komputera operacje te są kompletnie różne, ale człowiek chciałby mieć je pod tą samą nazwą.

W Haskellu mamy **polimorfizm parametryczny**, dzięki któremu możemy pisać funkcje dla typów ogólnych, a nie dla każdego możliwego typu z osobna, jak np. `length :: [a] -> Int`.

Haskell ma też inny rodzaj polimorfizmu, czyli **mechanizm klas**. Klasy w Haskellu różnią się jednak od znanych nam do tej pory odpowiedników w językach obiektowych, jak Java. O ile w Javie klasa jest bezpośrednio typem obiektu, o tyle w Haskellu są to bardziej typy z zestawem operacji. Bliżej im do abstrakcyjnych typów danych. Klasy umożliwiają nam wymaganie, żeby dany typ udostępniał nam pewne operacje.

    #!haskell
    class Monoid a where
        e :: a
        (++) :: a -> a -> a

        -- Aksjomaty, żeby było wiadomo, że działa jak powinno
        -- x ++ e = x
        -- e ++ x = x
        -- (x ++ y) ++ z = x ++ (y ++ z)

    instance Monoid Integer where
        e = 0
        (++) = (*)

        -- 0 * x = x
        -- x * 0 = x
        -- (x * y) * z = x * (y * z)

To wszystko umożliwia nam na wielki _code reuse_. Wystarczy nam jedna implementacja problemu dla danej klasy, żeby działało dla wielu typów tej klasy.

## Przykłady operacji, które mogłyby działać na monoidach

### Pobieranie głowy listy

Znana nam dobrze implementacja pobierania głowy listy wygląda tak:

    #!haskell
    head :: [a] -> a
    head (x:_) = x

Dla pustej listy wtedy jednak twardo wywali błąd i to nie jest złe. Przyjmijmy jednak, że bardzo byśmy chcieli, żeby nie wywaliło, możemy użyć typu `Maybe`:

    #!haskell
    data Maybe a = Just a | Nothing

To oznacza, że jeżeli funkcja zwraca `Maybe a`, to może zwrócić albo _po prostu_ `a` (`Just a`), albo nic (`Nothing`). Teraz moglibyśmy napisać `heada` tak:

    #!haskell
    head :: [a] -> Maybe a
    head (x:_) = Just x
    head [] = Nothing

Tyle, że wadą tego rozwiązania jest konieczność obsługi tego. O ile w pierwszej wersji mogliśmy napisać `(head xs + 1) :: Integer` i jak wybuchło, to nas to nie interesuje, o tyle teraz musimy zrobić tak:

    #!haskell
    (case head xs of
        Just x -> Just $ x+1
        Nothing -> Nothing) :: Maybe Integer

To nie jest zbyt wygodne, dlatego wyposażymy `Maybe` w operacje, które ułatwią nam życie.

    #!haskell
    data Maybe a = Just a | Nothing

    (>>=) :: Maybe a -> (a -> Maybe b) -> Maybe b
    return :: a -> Maybe a
    fail :: Maybe a

    return = Just
    fail = Nothing
    m >>= f = case m of
        Just a -> f a
        Nothing -> Nothing

Teraz możemy zrobić o wiele przyjemniejsze:

    #!haskell
    (head xs) >>= (\x -> Just $ x+1)

### Chcemy liczyć z nawrotami!

Podstawą liczenia z nawrotami jest generowanie permutacji. Istnieje niesamowite podobieństwo pomiędzy generowaniem permutacji w Prologu i w Haskellu.

    #!prolog
    % perm/2
    
    perm([], []).
    perm(X, [H|Z]) :-
        select(X, H, Y),
        perm(Y, Z).
^

    #!haskell
    perm :: [a] -> [[a]]

    perm [] = [[]]
    perm xs = select xs `concatMap` (\(x,ys) -> map (x:) (perm ys))

Obie implementacje właściwie znaczą to samo. O ile Prolog generuje nową permutację z każdym kolejnym nawrotem, to w Haskellu korzystamy z faktu, że funkcja `perm` leniwie generuje nam tyle permutacji, ile dokładnie potrzebujemy.

Okazuje się, że lista ma podobną strukturę do typu `Maybe`:

    #!haskell
    data [a] = (:) a [a] | []
    
    (>>=) :: [a] -> (a -> [b]) -> [b]
    return :: a -> [a]
    fail :: [a]

Wtedy moglibyśmy napisać permutacje tak:

    #!haskell
    perm [] = return []
    perm xs = select xs >>= (\ (x, ys) -> map (x:) (perm ys))

Istnieje notacja z `do`, która nam znakomicie upraszcza kod.

    #!haskell
    perm [] = return []
    perm xs = do
        (x, ys) <- select xs
        map (x:) (perm ys)

Widzimy tutaj, że o ile funkcje zwracające `Maybe` zwracały wynik albo nic, to funkcje zwracające listę zwracają wiele wyników. Zatem lista jest uogólnieniem Maybe.

### Szukanie elementu w liście

Bazując na tym, co zrobiliśmy, możemy zrobić funkcję szukającą elementu w liście.

    #!haskell
    find_maybe :: (a -> Bool) -> [a] -> Maybe a
    find_list :: (a -> Bool) -> [a] -> [a]

To jednak nie jest fajne. Fajnie byłoby mieć od tego jedną funkcję.

> _przykład finda_

### Numerowanie elementów drzewa

    #!haskell
    data Tree a = Node (Tree a) a (Tree a) | Tip

    renumber :: Tree a -> Tree Integer

    renumber = fst . aux 1 where
        aux n Tip = (Tip, n)
        aux n (Node l _ r) = (Node (l', n', r'), n'') where
            (l', n') = aux n l
            (r', n'') = aux (n'+1) r

Jak widać, ta funkcja jest _co najmniej_ niefajna. Jednak żeby fajnie ją przerobić, potrzebowalibyśmy jakiejś zmiennej globalnej. Okazuje się, że na ratunek przychodzi nam taka struktura: `StanPoczątkowy -> (Wynik, StanKońcowy)`. Okazuje się, że właśnie `aux` ma taką strukturę:

    #!haskell
    aux :: Tree a -> Integer -> (Tree Integer | Integer)

Zróbmy sobie zatem typ dla obliczenia i spróbujmy go zastosować:

    #!haskell
    type Comp s a = s -> (a, s)

    aux :: Tree a -> Comp Integer (Tree Integer)

Obliczenia można ułożyć w sekwencję i to tez zdefiniujemy:

    #!haskell
    return a -> Comp s a
    (>>=) :: Comp s a -> (a -> Comp s b) -> Comp s b

    -- obliczenie modyfikujące stan
    c >>= f = \s ->
        let
            (a, s') = c s
        in
            f a s'

    -- obliczenie niemodyfikujące stanu
    return a = \s -> (a, s)

Teraz możemy przepisać `renumber`:

    #!haskell
    plus_plus :: Comp Integer Integer
    plus_plus = \s -> (s, s+1)

    renumber t = fst . aux t 1 where
        aux Tip = return Tip
        aux (Node l _ r) = aux l >>= (\l' -> plus_plus >>= (\n -> aux r >>= (\r' -> return $ Node l' n r')))

    -- w notacji do

    renumber t = fst . aux t 1 where
        aux Tip = return Tip
        aux (Node l _ r) = do
            l' <- aux l
            n <- plus_plus
            r' <- aux r
            return $ Node l' n r'

Znowu mamy typ, który ma `return` i `>>=`. Zatem moglibiśmy zrobić klasę, która nam ujednolici podstawowe cechy wszystkich takich typów.

## Monada

    #!haskell
    class Monad (m :: x -> x) where
        (>>=) :: m a -> (a -> m b) -> m b
        return :: a -> m a

        -- (m >>= \x -> n) >>= f = m >>= (\x -> (n >>= f))
        -- return a >>= f = f a
        -- m >>= return = m

Widzimy tutaj coś bardzo podobnego do monoidu, ale jeszcze bardziej ogólnego.
