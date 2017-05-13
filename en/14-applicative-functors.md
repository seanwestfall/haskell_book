Haskell

Applicative functors

disregard this analogy
In this section, we'll take a look at applicative functors, which are beefed up functors, represented in Haskell by the Applicative typeclass, found in the Control.Applicative module.

As you know, functions in Haskell are curried by default, which means that a function that seems to take several parameters actually takes just one parameter and returns a function that takes the next parameter and so on. If a function is of type a -> b -> c, we usually say that it takes two parameters and returns a c, but actually it takes an a and returns a function b -> c. That's why we can call a function as f x y or as (f x) y. This mechanism is what enables us to partially apply functions by just calling them with too few parameters, which results in functions that we can then pass on to other functions.

So far, when we were mapping functions over functors, we usually mapped functions that take only one parameter. But what happens when we map a function like *, which takes two parameters, over a functor? Let's take a look at a couple of concrete examples of this. If we have Just 3 and we do fmap (*) (Just 3), what do we get? From the instance implementation of Maybe for Functor, we know that if it's a Just something value, it will apply the function to the something inside the Just. Therefore, doing fmap (*) (Just 3) results in Just ((*) 3), which can also be written as Just (* 3) if we use sections. Interesting! We get a function wrapped in a Just!

ghci> :t fmap (++) (Just "hey")  
fmap (++) (Just "hey") :: Maybe ([Char] -> [Char])  
ghci> :t fmap compare (Just 'a')  
fmap compare (Just 'a') :: Maybe (Char -> Ordering)  
ghci> :t fmap compare "A LIST OF CHARS"  
fmap compare "A LIST OF CHARS" :: [Char -> Ordering]  
ghci> :t fmap (\x y z -> x + y / z) [3,4,5,6]  
fmap (\x y z -> x + y / z) [3,4,5,6] :: (Fractional a) => [a -> a -> a]  
If we map compare, which has a type of (Ord a) => a -> a -> Ordering over a list of characters, we get a list of functions of type Char -> Ordering, because the function compare gets partially applied with the characters in the list. It's not a list of (Ord a) => a -> Ordering function, because the first a that got applied was a Char and so the second a has to decide to be of type Char.

We see how by mapping "multi-parameter" functions over functors, we get functors that contain functions inside them. So now what can we do with them? Well for one, we can map functions that take these functions as parameters over them, because whatever is inside a functor will be given to the function that we're mapping over it as a parameter.

ghci> let a = fmap (*) [1,2,3,4]  
ghci> :t a  
a :: [Integer -> Integer]  
ghci> fmap (\f -> f 9) a  
[9,18,27,36]  
But what if we have a functor value of Just (3 *) and a functor value of Just 5 and we want to take out the function from Just (3 *) and map it over Just 5? With normal functors, we're out of luck, because all they support is just mapping normal functions over existing functors. Even when we mapped \f -> f 9 over a functor that contained functions inside it, we were just mapping a normal function over it. But we can't map a function that's inside a functor over another functor with what fmap offers us. We could pattern-match against the Just constructor to get the function out of it and then map it over Just 5, but we're looking for a more general and abstract way of doing that, which works across functors.

Meet the Applicative typeclass. It lies in the Control.Applicative module and it defines two methods, pure and <*>. It doesn't provide a default implementation for any of them, so we have to define them both if we want something to be an applicative functor. The class is defined like so:

class (Functor f) => Applicative f where  
    pure :: a -> f a  
    (<*>) :: f (a -> b) -> f a -> f b  
This simple three line class definition tells us a lot! Let's start at the first line. It starts the definition of the Applicative class and it also introduces a class constraint. It says that if we want to make a type constructor part of the Applicative typeclass, it has to be in Functor first. That's why if we know that if a type constructor is part of the Applicative typeclass, it's also in Functor, so we can use fmap on it.

The first method it defines is called pure. Its type declaration is pure :: a -> f a. f plays the role of our applicative functor instance here. Because Haskell has a very good type system and because everything a function can do is take some parameters and return some value, we can tell a lot from a type declaration and this is no exception. pure should take a value of any type and return an applicative functor with that value inside it. When we say inside it, we're using the box analogy again, even though we've seen that it doesn't always stand up to scrutiny. But the a -> f a type declaration is still pretty descriptive. We take a value and we wrap it in an applicative functor that has that value as the result inside it.

A better way of thinking about pure would be to say that it takes a value and puts it in some sort of default (or pure) context—a minimal context that still yields that value.

The <*> function is really interesting. It has a type declaration of f (a -> b) -> f a -> f b. Does this remind you of anything? Of course, fmap :: (a -> b) -> f a -> f b. It's a sort of a beefed up fmap. Whereas fmap takes a function and a functor and applies the function inside the functor, <*> takes a functor that has a function in it and another functor and sort of extracts that function from the first functor and then maps it over the second one. When I say extract, I actually sort of mean run and then extract, maybe even sequence. We'll see why soon.

Let's take a look at the Applicative instance implementation for Maybe.

instance Applicative Maybe where  
    pure = Just  
    Nothing <*> _ = Nothing  
    (Just f) <*> something = fmap f something  
Again, from the class definition we see that the f that plays the role of the applicative functor should take one concrete type as a parameter, so we write instance Applicative Maybe where instead of writing instance Applicative (Maybe a) where.

First off, pure. We said earlier that it's supposed to take something and wrap it in an applicative functor. We wrote pure = Just, because value constructors like Just are normal functions. We could have also written pure x = Just x.

Next up, we have the definition for <*>. We can't extract a function out of a Nothing, because it has no function inside it. So we say that if we try to extract a function from a Nothing, the result is a Nothing. If you look at the class definition for Applicative, you'll see that there's a Functor class constraint, which means that we can assume that both of <*>'s parameters are functors. If the first parameter is not a Nothing, but a Just with some function inside it, we say that we then want to map that function over the second parameter. This also takes care of the case where the second parameter is Nothing, because doing fmap with any function over a Nothing will return a Nothing.

So for Maybe, <*> extracts the function from the left value if it's a Just and maps it over the right value. If any of the parameters is Nothing, Nothing is the result.

OK cool great. Let's give this a whirl.

ghci> Just (+3) <*> Just 9  
Just 12  
ghci> pure (+3) <*> Just 10  
Just 13  
ghci> pure (+3) <*> Just 9  
Just 12  
ghci> Just (++"hahah") <*> Nothing  
Nothing  
ghci> Nothing <*> Just "woot"  
Nothing  
We see how doing pure (+3) and Just (+3) is the same in this case. Use pure if you're dealing with Maybe values in an applicative context (i.e. using them with <*>), otherwise stick to Just. The first four input lines demonstrate how the function is extracted and then mapped, but in this case, they could have been achieved by just mapping unwrapped functions over functors. The last line is interesting, because we try to extract a function from a Nothing and then map it over something, which of course results in a Nothing.

With normal functors, you can just map a function over a functor and then you can't get the result out in any general way, even if the result is a partially applied function. Applicative functors, on the other hand, allow you to operate on several functors with a single function. Check out this piece of code:

ghci> pure (+) <*> Just 3 <*> Just 5  
Just 8  
ghci> pure (+) <*> Just 3 <*> Nothing  
Nothing  
ghci> pure (+) <*> Nothing <*> Just 5  
Nothing  
whaale
What's going on here? Let's take a look, step by step. <*> is left-associative, which means that pure (+) <*> Just 3 <*> Just 5 is the same as (pure (+) <*> Just 3) <*> Just 5. First, the + function is put in a functor, which is in this case a Maybe value that contains the function. So at first, we have pure (+), which is Just (+). Next, Just (+) <*> Just 3 happens. The result of this is Just (3+). This is because of partial application. Only applying 3 to the + function results in a function that takes one parameter and adds 3 to it. Finally, Just (3+) <*> Just 5 is carried out, which results in a Just 8.

Isn't this awesome?! Applicative functors and the applicative style of doing pure f <*> x <*> y <*> ... allow us to take a function that expects parameters that aren't necessarily wrapped in functors and use that function to operate on several values that are in functor contexts. The function can take as many parameters as we want, because it's always partially applied step by step between occurences of <*>.

This becomes even more handy and apparent if we consider the fact that pure f <*> x equals fmap f x. This is one of the applicative laws. We'll take a closer look at them later, but for now, we can sort of intuitively see that this is so. Think about it, it makes sense. Like we said before, pure puts a value in a default context. If we just put a function in a default context and then extract and apply it to a value inside another applicative functor, we did the same as just mapping that function over that applicative functor. Instead of writing pure f <*> x <*> y <*> ..., we can write fmap f x <*> y <*> .... This is why Control.Applicative exports a function called <$>, which is just fmap as an infix operator. Here's how it's defined:

(<$>) :: (Functor f) => (a -> b) -> f a -> f b  
f <$> x = fmap f x  
Yo! Quick reminder: type variables are independent of parameter names or other value names. The f in the function declaration here is a type variable with a class constraint saying that any type constructor that replaces f should be in the Functor typeclass. The f in the function body denotes a function that we map over x. The fact that we used f to represent both of those doesn't mean that they somehow represent the same thing.
By using <$>, the applicative style really shines, because now if we want to apply a function f between three applicative functors, we can write f <$> x <*> y <*> z. If the parameters weren't applicative functors but normal values, we'd write f x y z.

Let's take a closer look at how this works. We have a value of Just "johntra" and a value of Just "volta" and we want to join them into one String inside a Maybe functor. We do this:

ghci> (++) <$> Just "johntra" <*> Just "volta"  
Just "johntravolta"  
Before we see how this happens, compare the above line with this:

ghci> (++) "johntra" "volta"  
"johntravolta"  
Awesome! To use a normal function on applicative functors, just sprinkle some <$> and <*> about and the function will operate on applicatives and return an applicative. How cool is that?

Anyway, when we do (++) <$> Just "johntra" <*> Just "volta", first (++), which has a type of (++) :: [a] -> [a] -> [a] gets mapped over Just "johntra", resulting in a value that's the same as Just ("johntra"++) and has a type of Maybe ([Char] -> [Char]). Notice how the first parameter of (++) got eaten up and how the as turned into Chars. And now Just ("johntra"++) <*> Just "volta" happens, which takes the function out of the Just and maps it over Just "volta", resulting in Just "johntravolta". Had any of the two values been Nothing, the result would have also been Nothing.

So far, we've only used Maybe in our examples and you might be thinking that applicative functors are all about Maybe. There are loads of other instances of Applicative, so let's go and meet them!

Lists (actually the list type constructor, []) are applicative functors. What a suprise! Here's how [] is an instance of Applicative:

instance Applicative [] where  
    pure x = [x]  
    fs <*> xs = [f x | f <- fs, x <- xs]  
Earlier, we said that pure takes a value and puts it in a default context. Or in other words, a minimal context that still yields that value. The minimal context for lists would be the empty list, [], but the empty list represents the lack of a value, so it can't hold in itself the value that we used pure on. That's why pure takes a value and puts it in a singleton list. Similarly, the minimal context for the Maybe applicative functor would be a Nothing, but it represents the lack of a value instead of a value, so pure is implemented as Just in the instance implementation for Maybe.

ghci> pure "Hey" :: [String]  
["Hey"]  
ghci> pure "Hey" :: Maybe String  
Just "Hey"  
What about <*>? If we look at what <*>'s type would be if it were limited only to lists, we get (<*>) :: [a -> b] -> [a] -> [b]. It's implemented with a list comprehension. <*> has to somehow extract the function out of its left parameter and then map it over the right parameter. But the thing here is that the left list can have zero functions, one function, or several functions inside it. The right list can also hold several values. That's why we use a list comprehension to draw from both lists. We apply every possible function from the left list to every possible value from the right list. The resulting list has every possible combination of applying a function from the left list to a value in the right one.

ghci> [(*0),(+100),(^2)] <*> [1,2,3]  
[0,0,0,101,102,103,1,4,9]  
The left list has three functions and the right list has three values, so the resulting list will have nine elements. Every function in the left list is applied to every function in the right one. If we have a list of functions that take two parameters, we can apply those functions between two lists.

ghci> [(+),(*)] <*> [1,2] <*> [3,4]  
[4,5,5,6,3,4,6,8]  
Because <*> is left-associative, [(+),(*)] <*> [1,2] happens first, resulting in a list that's the same as [(1+),(2+),(1*),(2*)], because every function on the left gets applied to every value on the right. Then, [(1+),(2+),(1*),(2*)] <*> [3,4] happens, which produces the final result.

Using the applicative style with lists is fun! Watch:

ghci> (++) <$> ["ha","heh","hmm"] <*> ["?","!","."]  
["ha?","ha!","ha.","heh?","heh!","heh.","hmm?","hmm!","hmm."]  
Again, see how we used a normal function that takes two strings between two applicative functors of strings just by inserting the appropriate applicative operators.

You can view lists as non-deterministic computations. A value like 100 or "what" can be viewed as a deterministic computation that has only one result, whereas a list like [1,2,3] can be viewed as a computation that can't decide on which result it wants to have, so it presents us with all of the possible results. So when you do something like (+) <$> [1,2,3] <*> [4,5,6], you can think of it as adding together two non-deterministic computations with +, only to produce another non-deterministic computation that's even less sure about its result.

Using the applicative style on lists is often a good replacement for list comprehensions. In the second chapter, we wanted to see all the possible products of [2,5,10] and [8,10,11], so we did this:

ghci> [ x*y | x <- [2,5,10], y <- [8,10,11]]     
[16,20,22,40,50,55,80,100,110]     
We're just drawing from two lists and applying a function between every combination of elements. This can be done in the applicative style as well:

ghci> (*) <$> [2,5,10] <*> [8,10,11]  
[16,20,22,40,50,55,80,100,110]  
This seems clearer to me, because it's easier to see that we're just calling * between two non-deterministic computations. If we wanted all possible products of those two lists that are more than 50, we'd just do:

ghci> filter (>50) $ (*) <$> [2,5,10] <*> [8,10,11]  
[55,80,100,110]  
It's easy to see how pure f <*> xs equals fmap f xs with lists. pure f is just [f] and [f] <*> xs will apply every function in the left list to every value in the right one, but there's just one function in the left list, so it's like mapping.

Another instance of Applicative that we've already encountered is IO. This is how the instance is implemented:

instance Applicative IO where  
    pure = return  
    a <*> b = do  
        f <- a  
        x <- b  
        return (f x)  
ahahahah!
Since pure is all about putting a value in a minimal context that still holds it as its result, it makes sense that pure is just return, because return does exactly that; it makes an I/O action that doesn't do anything, it just yields some value as its result, but it doesn't really do any I/O operations like printing to the terminal or reading from a file.

If <*> were specialized for IO it would have a type of (<*>) :: IO (a -> b) -> IO a -> IO b. It would take an I/O action that yields a function as its result and another I/O action and create a new I/O action from those two that, when performed, first performs the first one to get the function and then performs the second one to get the value and then it would yield that function applied to the value as its result. We used do syntax to implement it here. Remember, do syntax is about taking several I/O actions and gluing them into one, which is exactly what we do here.

With Maybe and [], we could think of <*> as simply extracting a function from its left parameter and then sort of applying it over the right one. With IO, extracting is still in the game, but now we also have a notion of sequencing, because we're taking two I/O actions and we're sequencing, or gluing, them into one. We have to extract the function from the first I/O action, but to extract a result from an I/O action, it has to be performed.

Consider this:

myAction :: IO String  
myAction = do  
    a <- getLine  
    b <- getLine  
    return $ a ++ b  
This is an I/O action that will prompt the user for two lines and yield as its result those two lines concatenated. We achieved it by gluing together two getLine I/O actions and a return, because we wanted our new glued I/O action to hold the result of a ++ b. Another way of writing this would be to use the applicative style.

myAction :: IO String  
myAction = (++) <$> getLine <*> getLine  
What we were doing before was making an I/O action that applied a function between the results of two other I/O actions, and this is the same thing. Remember, getLine is an I/O action with the type getLine :: IO String. When we use <*> between two applicative functors, the result is an applicative functor, so this all makes sense.

If we regress to the box analogy, we can imagine getLine as a box that will go out into the real world and fetch us a string. Doing (++) <$> getLine <*> getLine makes a new, bigger box that sends those two boxes out to fetch lines from the terminal and then presents the concatenation of those two lines as its result.

The type of the expression (++) <$> getLine <*> getLine is IO String, which means that this expression is a completely normal I/O action like any other, which also holds a result value inside it, just like other I/O actions. That's why we can do stuff like:

main = do  
    a <- (++) <$> getLine <*> getLine  
    putStrLn $ "The two lines concatenated turn out to be: " ++ a  
If you ever find yourself binding some I/O actions to names and then calling some function on them and presenting that as the result by using return, consider using the applicative style because it's arguably a bit more concise and terse.

Another instance of Applicative is (->) r, so functions. They are rarely used with the applicative style outside of code golf, but they're still interesting as applicatives, so let's take a look at how the function instance is implemented.

If you're confused about what (->) r means, check out the previous section where we explain how (->) r is a functor.
instance Applicative ((->) r) where  
    pure x = (\_ -> x)  
    f <*> g = \x -> f x (g x)  
When we wrap a value into an applicative functor with pure, the result it yields always has to be that value. A minimal default context that still yields that value as a result. That's why in the function instance implementation, pure takes a value and creates a function that ignores its parameter and always returns that value. If we look at the type for pure, but specialized for the (->) r instance, it's pure :: a -> (r -> a).

ghci> (pure 3) "blah"  
3  
Because of currying, function application is left-associative, so we can omit the parentheses.

ghci> pure 3 "blah"  
3  
The instance implementation for <*> is a bit cryptic, so it's best if we just take a look at how to use functions as applicative functors in the applicative style.

ghci> :t (+) <$> (+3) <*> (*100)  
(+) <$> (+3) <*> (*100) :: (Num a) => a -> a  
ghci> (+) <$> (+3) <*> (*100) $ 5  
508  
Calling <*> with two applicative functors results in an applicative functor, so if we use it on two functions, we get back a function. So what goes on here? When we do (+) <$> (+3) <*> (*100), we're making a function that will use + on the results of (+3) and (*100) and return that. To demonstrate on a real example, when we did (+) <$> (+3) <*> (*100) $ 5, the 5 first got applied to (+3) and (*100), resulting in 8 and 500. Then, + gets called with 8 and 500, resulting in 508.

ghci> (\x y z -> [x,y,z]) <$> (+3) <*> (*2) <*> (/2) $ 5  
[8.0,10.0,2.5]  
SLAP
Same here. We create a function that will call the function \x y z -> [x,y,z] with the eventual results from (+3), (*2) and (/2). The 5 gets fed to each of the three functions and then \x y z -> [x, y, z] gets called with those results.

You can think of functions as boxes that contain their eventual results, so doing k <$> f <*> g creates a function that will call k with the eventual results from f and g. When we do something like (+) <$> Just 3 <*> Just 5, we're using + on values that might or might not be there, which also results in a value that might or might not be there. When we do (+) <$> (+10) <*> (+5), we're using + on the future return values of (+10) and (+5) and the result is also something that will produce a value only when called with a parameter.

We don't often use functions as applicatives, but this is still really interesting. It's not very important that you get how the (->) r instance for Applicative works, so don't despair if you're not getting this right now. Try playing with the applicative style and functions to build up an intuition for functions as applicatives.

An instance of Applicative that we haven't encountered yet is ZipList, and it lives in Control.Applicative.

It turns out there are actually more ways for lists to be applicative functors. One way is the one we already covered, which says that calling <*> with a list of functions and a list of values results in a list which has all the possible combinations of applying functions from the left list to the values in the right list. If we do [(+3),(*2)] <*> [1,2], (+3) will be applied to both 1 and 2 and (*2) will also be applied to both 1 and 2, resulting in a list that has four elements, namely [4,5,2,4].

However, [(+3),(*2)] <*> [1,2] could also work in such a way that the first function in the left list gets applied to the first value in the right one, the second function gets applied to the second value, and so on. That would result in a list with two values, namely [4,4]. You could look at it as [1 + 3, 2 * 2].

Because one type can't have two instances for the same typeclass, the ZipList a type was introduced, which has one constructor ZipList that has just one field, and that field is a list. Here's the instance:

instance Applicative ZipList where  
        pure x = ZipList (repeat x)  
        ZipList fs <*> ZipList xs = ZipList (zipWith (\f x -> f x) fs xs)  
<*> does just what we said. It applies the first function to the first value, the second function to the second value, etc. This is done with zipWith (\f x -> f x) fs xs. Because of how zipWith works, the resulting list will be as long as the shorter of the two lists.

pure is also interesting here. It takes a value and puts it in a list that just has that value repeating indefinitely. pure "haha" results in ZipList (["haha","haha","haha".... This might be a bit confusing since we said that pure should put a value in a minimal context that still yields that value. And you might be thinking that an infinite list of something is hardly minimal. But it makes sense with zip lists, because it has to produce the value on every position. This also satisfies the law that pure f <*> xs should equal fmap f xs. If pure 3 just returned ZipList [3], pure (*2) <*> ZipList [1,5,10] would result in ZipList [2], because the resulting list of two zipped lists has the length of the shorter of the two. If we zip a finite list with an infinite list, the length of the resulting list will always be equal to the length of the finite list.

So how do zip lists work in an applicative style? Let's see. Oh, the ZipList a type doesn't have a Show instance, so we have to use the getZipList function to extract a raw list out of a zip list.

ghci> getZipList $ (+) <$> ZipList [1,2,3] <*> ZipList [100,100,100]  
[101,102,103]  
ghci> getZipList $ (+) <$> ZipList [1,2,3] <*> ZipList [100,100..]  
[101,102,103]  
ghci> getZipList $ max <$> ZipList [1,2,3,4,5,3] <*> ZipList [5,3,1,2]  
[5,3,3,4]  
ghci> getZipList $ (,,) <$> ZipList "dog" <*> ZipList "cat" <*> ZipList "rat"  
[('d','c','r'),('o','a','a'),('g','t','t')]  
The (,,) function is the same as \x y z -> (x,y,z). Also, the (,) function is the same as \x y -> (x,y).
Aside from zipWith, the standard library has functions such as zipWith3, zipWith4, all the way up to 7. zipWith takes a function that takes two parameters and zips two lists with it. zipWith3 takes a function that takes three parameters and zips three lists with it, and so on. By using zip lists with an applicative style, we don't have to have a separate zip function for each number of lists that we want to zip together. We just use the applicative style to zip together an arbitrary amount of lists with a function, and that's pretty cool.

Control.Applicative defines a function that's called liftA2, which has a type of liftA2 :: (Applicative f) => (a -> b -> c) -> f a -> f b -> f c . It's defined like this:

liftA2 :: (Applicative f) => (a -> b -> c) -> f a -> f b -> f c  
liftA2 f a b = f <$> a <*> b  
Nothing special, it just applies a function between two applicatives, hiding the applicative style that we've become familiar with. The reason we're looking at it is because it clearly showcases why applicative functors are more powerful than just ordinary functors. With ordinary functors, we can just map functions over one functor. But with applicative functors, we can apply a function between several functors. It's also interesting to look at this function's type as (a -> b -> c) -> (f a -> f b -> f c). When we look at it like this, we can say that liftA2 takes a normal binary function and promotes it to a function that operates on two functors.

Here's an interesting concept: we can take two applicative functors and combine them into one applicative functor that has inside it the results of those two applicative functors in a list. For instance, we have Just 3 and Just 4. Let's assume that the second one has a singleton list inside it, because that's really easy to achieve:

ghci> fmap (\x -> [x]) (Just 4)  
Just [4]  
OK, so let's say we have Just 3 and Just [4]. How do we get Just [3,4]? Easy.

ghci> liftA2 (:) (Just 3) (Just [4])  
Just [3,4]  
ghci> (:) <$> Just 3 <*> Just [4]  
Just [3,4]  
Remember, : is a function that takes an element and a list and returns a new list with that element at the beginning. Now that we have Just [3,4], could we combine that with Just 2 to produce Just [2,3,4]? Of course we could. It seems that we can combine any amount of applicatives into one applicative that has a list of the results of those applicatives inside it. Let's try implementing a function that takes a list of applicatives and returns an applicative that has a list as its result value. We'll call it sequenceA.

sequenceA :: (Applicative f) => [f a] -> f [a]  
sequenceA [] = pure []  
sequenceA (x:xs) = (:) <$> x <*> sequenceA xs  
Ah, recursion! First, we look at the type. It will transform a list of applicatives into an applicative with a list. From that, we can lay some groundwork for an edge condition. If we want to turn an empty list into an applicative with a list of results, well, we just put an empty list in a default context. Now comes the recursion. If we have a list with a head and a tail (remember, x is an applicative and xs is a list of them), we call sequenceA on the tail, which results in an applicative with a list. Then, we just prepend the value inside the applicative x into that applicative with a list, and that's it!

So if we do sequenceA [Just 1, Just 2], that's (:) <$> Just 1 <*> sequenceA [Just 2] . That equals (:) <$> Just 1 <*> ((:) <$> Just 2 <*> sequenceA []). Ah! We know that sequenceA [] ends up as being Just [], so this expression is now (:) <$> Just 1 <*> ((:) <$> Just 2 <*> Just []), which is (:) <$> Just 1 <*> Just [2], which is Just [1,2]!

Another way to implement sequenceA is with a fold. Remember, pretty much any function where we go over a list element by element and accumulate a result along the way can be implemented with a fold.

sequenceA :: (Applicative f) => [f a] -> f [a]  
sequenceA = foldr (liftA2 (:)) (pure [])  
We approach the list from the right and start off with an accumulator value of pure []. We do liftA2 (:) between the accumulator and the last element of the list, which results in an applicative that has a singleton in it. Then we do liftA2 (:) with the now last element and the current accumulator and so on, until we're left with just the accumulator, which holds a list of the results of all the applicatives.

Let's give our function a whirl on some applicatives.

ghci> sequenceA [Just 3, Just 2, Just 1]  
Just [3,2,1]  
ghci> sequenceA [Just 3, Nothing, Just 1]  
Nothing  
ghci> sequenceA [(+3),(+2),(+1)] 3  
[6,5,4]  
ghci> sequenceA [[1,2,3],[4,5,6]]  
[[1,4],[1,5],[1,6],[2,4],[2,5],[2,6],[3,4],[3,5],[3,6]]  
ghci> sequenceA [[1,2,3],[4,5,6],[3,4,4],[]]  
[]  
Ah! Pretty cool. When used on Maybe values, sequenceA creates a Maybe value with all the results inside it as a list. If one of the values was Nothing, then the result is also a Nothing. This is cool when you have a list of Maybe values and you're interested in the values only if none of them is a Nothing.

When used with functions, sequenceA takes a list of functions and returns a function that returns a list. In our example, we made a function that took a number as a parameter and applied it to each function in the list and then returned a list of results. sequenceA [(+3),(+2),(+1)] 3 will call (+3) with 3, (+2) with 3 and (+1) with 3 and present all those results as a list.

Doing (+) <$> (+3) <*> (*2) will create a function that takes a parameter, feeds it to both (+3) and (*2) and then calls + with those two results. In the same vein, it makes sense that sequenceA [(+3),(*2)] makes a function that takes a parameter and feeds it to all of the functions in the list. Instead of calling + with the results of the functions, a combination of : and pure [] is used to gather those results in a list, which is the result of that function.

Using sequenceA is cool when we have a list of functions and we want to feed the same input to all of them and then view the list of results. For instance, we have a number and we're wondering whether it satisfies all of the predicates in a list. One way to do that would be like so:

ghci> map (\f -> f 7) [(>4),(<10),odd]  
[True,True,True]  
ghci> and $ map (\f -> f 7) [(>4),(<10),odd]  
True  
Remember, and takes a list of booleans and returns True if they're all True. Another way to achieve the same thing would be with sequenceA:

ghci> sequenceA [(>4),(<10),odd] 7  
[True,True,True]  
ghci> and $ sequenceA [(>4),(<10),odd] 7  
True  
sequenceA [(>4),(<10),odd] creates a function that will take a number and feed it to all of the predicates in [(>4),(<10),odd] and return a list of booleans. It turns a list with the type (Num a) => [a -> Bool] into a function with the type (Num a) => a -> [Bool]. Pretty neat, huh?

Because lists are homogenous, all the functions in the list have to be functions of the same type, of course. You can't have a list like [ord, (+3)], because ord takes a character and returns a number, whereas (+3) takes a number and returns a number.

When used with [], sequenceA takes a list of lists and returns a list of lists. Hmm, interesting. It actually creates lists that have all possible combinations of their elements. For illustration, here's the above done with sequenceA and then done with a list comprehension:

ghci> sequenceA [[1,2,3],[4,5,6]]  
[[1,4],[1,5],[1,6],[2,4],[2,5],[2,6],[3,4],[3,5],[3,6]]  
ghci> [[x,y] | x <- [1,2,3], y <- [4,5,6]]  
[[1,4],[1,5],[1,6],[2,4],[2,5],[2,6],[3,4],[3,5],[3,6]]  
ghci> sequenceA [[1,2],[3,4]]  
[[1,3],[1,4],[2,3],[2,4]]  
ghci> [[x,y] | x <- [1,2], y <- [3,4]]  
[[1,3],[1,4],[2,3],[2,4]]  
ghci> sequenceA [[1,2],[3,4],[5,6]]  
[[1,3,5],[1,3,6],[1,4,5],[1,4,6],[2,3,5],[2,3,6],[2,4,5],[2,4,6]]  
ghci> [[x,y,z] | x <- [1,2], y <- [3,4], z <- [5,6]]  
[[1,3,5],[1,3,6],[1,4,5],[1,4,6],[2,3,5],[2,3,6],[2,4,5],[2,4,6]]  
This might be a bit hard to grasp, but if you play with it for a while, you'll see how it works. Let's say that we're doing sequenceA [[1,2],[3,4]]. To see how this happens, let's use the sequenceA (x:xs) = (:) <$> x <*> sequenceA xs definition of sequenceA and the edge condition sequenceA [] = pure []. You don't have to follow this evaluation, but it might help you if have trouble imagining how sequenceA works on lists of lists, because it can be a bit mind-bending.

We start off with sequenceA [[1,2],[3,4]]
That evaluates to (:) <$> [1,2] <*> sequenceA [[3,4]]
Evaluating the inner sequenceA further, we get (:) <$> [1,2] <*> ((:) <$> [3,4] <*> sequenceA [])
We've reached the edge condition, so this is now (:) <$> [1,2] <*> ((:) <$> [3,4] <*> [[]])
Now, we evaluate the (:) <$> [3,4] <*> [[]] part, which will use : with every possible value in the left list (possible values are 3 and 4) with every possible value on the right list (only possible value is []), which results in [3:[], 4:[]], which is [[3],[4]]. So now we have (:) <$> [1,2] <*> [[3],[4]]
Now, : is used with every possible value from the left list (1 and 2) with every possible value in the right list ([3] and [4]), which results in [1:[3], 1:[4], 2:[3], 2:[4]], which is [[1,3],[1,4],[2,3],[2,4]
Doing (+) <$> [1,2] <*> [4,5,6]results in a non-deterministic computation x + y where x takes on every value from [1,2] and y takes on every value from [4,5,6]. We represent that as a list which holds all of the possible results. Similarly, when we do sequence [[1,2],[3,4],[5,6],[7,8]], the result is a non-deterministic computation [x,y,z,w], where x takes on every value from [1,2], y takes on every value from [3,4] and so on. To represent the result of that non-deterministic computation, we use a list, where each element in the list is one possible list. That's why the result is a list of lists.

When used with I/O actions, sequenceA is the same thing as sequence! It takes a list of I/O actions and returns an I/O action that will perform each of those actions and have as its result a list of the results of those I/O actions. That's because to turn an [IO a] value into an IO [a] value, to make an I/O action that yields a list of results when performed, all those I/O actions have to be sequenced so that they're then performed one after the other when evaluation is forced. You can't get the result of an I/O action without performing it.

ghci> sequenceA [getLine, getLine, getLine]  
heyh  
ho  
woo  
["heyh","ho","woo"]  
Like normal functors, applicative functors come with a few laws. The most important one is the one that we already mentioned, namely that pure f <*> x = fmap f x holds. As an exercise, you can prove this law for some of the applicative functors that we've met in this chapter.The other functor laws are:

pure id <*> v = v
pure (.) <*> u <*> v <*> w = u <*> (v <*> w)
pure f <*> pure x = pure (f x)
u <*> pure y = pure ($ y) <*> u
We won't go over them in detail right now because that would take up a lot of pages and it would probably be kind of boring, but if you're up to the task, you can take a closer look at them and see if they hold for some of the instances.

In conclusion, applicative functors aren't just interesting, they're also useful, because they allow us to combine different computations, such as I/O computations, non-deterministic computations, computations that might have failed, etc. by using the applicative style. Just by using <$> and <*> we can use normal functions to uniformly operate on any number of applicative functors and take advantage of the semantics of each one.

Purescript

Applicative functors

disregard this analogy
In this section, we'll take a look at applicative functors, which are beefed up functors, represented in Haskell by the Applicative typeclass, found in the Control.Applicative module.

As you know, functions in Haskell are curried by default, which means that a function that seems to take several parameters actually takes just one parameter and returns a function that takes the next parameter and so on. If a function is of type a -> b -> c, we usually say that it takes two parameters and returns a c, but actually it takes an a and returns a function b -> c. That's why we can call a function as f x y or as (f x) y. This mechanism is what enables us to partially apply functions by just calling them with too few parameters, which results in functions that we can then pass on to other functions.

So far, when we were mapping functions over functors, we usually mapped functions that take only one parameter. But what happens when we map a function like *, which takes two parameters, over a functor? Let's take a look at a couple of concrete examples of this. If we have Just 3 and we do fmap (*) (Just 3), what do we get? From the instance implementation of Maybe for Functor, we know that if it's a Just something value, it will apply the function to the something inside the Just. Therefore, doing fmap (*) (Just 3) results in Just ((*) 3), which can also be written as Just (* 3) if we use sections. Interesting! We get a function wrapped in a Just!

ghci> :t fmap (++) (Just "hey")  
fmap (++) (Just "hey") :: Maybe ([Char] -> [Char])  
ghci> :t fmap compare (Just 'a')  
fmap compare (Just 'a') :: Maybe (Char -> Ordering)  
ghci> :t fmap compare "A LIST OF CHARS"  
fmap compare "A LIST OF CHARS" :: [Char -> Ordering]  
ghci> :t fmap (\x y z -> x + y / z) [3,4,5,6]  
fmap (\x y z -> x + y / z) [3,4,5,6] :: (Fractional a) => [a -> a -> a]  
If we map compare, which has a type of (Ord a) => a -> a -> Ordering over a list of characters, we get a list of functions of type Char -> Ordering, because the function compare gets partially applied with the characters in the list. It's not a list of (Ord a) => a -> Ordering function, because the first a that got applied was a Char and so the second a has to decide to be of type Char.

We see how by mapping "multi-parameter" functions over functors, we get functors that contain functions inside them. So now what can we do with them? Well for one, we can map functions that take these functions as parameters over them, because whatever is inside a functor will be given to the function that we're mapping over it as a parameter.

ghci> let a = fmap (*) [1,2,3,4]  
ghci> :t a  
a :: [Integer -> Integer]  
ghci> fmap (\f -> f 9) a  
[9,18,27,36]  
But what if we have a functor value of Just (3 *) and a functor value of Just 5 and we want to take out the function from Just (3 *) and map it over Just 5? With normal functors, we're out of luck, because all they support is just mapping normal functions over existing functors. Even when we mapped \f -> f 9 over a functor that contained functions inside it, we were just mapping a normal function over it. But we can't map a function that's inside a functor over another functor with what fmap offers us. We could pattern-match against the Just constructor to get the function out of it and then map it over Just 5, but we're looking for a more general and abstract way of doing that, which works across functors.

Meet the Applicative typeclass. It lies in the Control.Applicative module and it defines two methods, pure and <*>. It doesn't provide a default implementation for any of them, so we have to define them both if we want something to be an applicative functor. The class is defined like so:

class (Functor f) => Applicative f where  
    pure :: a -> f a  
    (<*>) :: f (a -> b) -> f a -> f b  
This simple three line class definition tells us a lot! Let's start at the first line. It starts the definition of the Applicative class and it also introduces a class constraint. It says that if we want to make a type constructor part of the Applicative typeclass, it has to be in Functor first. That's why if we know that if a type constructor is part of the Applicative typeclass, it's also in Functor, so we can use fmap on it.

The first method it defines is called pure. Its type declaration is pure :: a -> f a. f plays the role of our applicative functor instance here. Because Haskell has a very good type system and because everything a function can do is take some parameters and return some value, we can tell a lot from a type declaration and this is no exception. pure should take a value of any type and return an applicative functor with that value inside it. When we say inside it, we're using the box analogy again, even though we've seen that it doesn't always stand up to scrutiny. But the a -> f a type declaration is still pretty descriptive. We take a value and we wrap it in an applicative functor that has that value as the result inside it.

A better way of thinking about pure would be to say that it takes a value and puts it in some sort of default (or pure) context—a minimal context that still yields that value.

The <*> function is really interesting. It has a type declaration of f (a -> b) -> f a -> f b. Does this remind you of anything? Of course, fmap :: (a -> b) -> f a -> f b. It's a sort of a beefed up fmap. Whereas fmap takes a function and a functor and applies the function inside the functor, <*> takes a functor that has a function in it and another functor and sort of extracts that function from the first functor and then maps it over the second one. When I say extract, I actually sort of mean run and then extract, maybe even sequence. We'll see why soon.

Let's take a look at the Applicative instance implementation for Maybe.

instance Applicative Maybe where  
    pure = Just  
    Nothing <*> _ = Nothing  
    (Just f) <*> something = fmap f something  
Again, from the class definition we see that the f that plays the role of the applicative functor should take one concrete type as a parameter, so we write instance Applicative Maybe where instead of writing instance Applicative (Maybe a) where.

First off, pure. We said earlier that it's supposed to take something and wrap it in an applicative functor. We wrote pure = Just, because value constructors like Just are normal functions. We could have also written pure x = Just x.

Next up, we have the definition for <*>. We can't extract a function out of a Nothing, because it has no function inside it. So we say that if we try to extract a function from a Nothing, the result is a Nothing. If you look at the class definition for Applicative, you'll see that there's a Functor class constraint, which means that we can assume that both of <*>'s parameters are functors. If the first parameter is not a Nothing, but a Just with some function inside it, we say that we then want to map that function over the second parameter. This also takes care of the case where the second parameter is Nothing, because doing fmap with any function over a Nothing will return a Nothing.

So for Maybe, <*> extracts the function from the left value if it's a Just and maps it over the right value. If any of the parameters is Nothing, Nothing is the result.

OK cool great. Let's give this a whirl.

ghci> Just (+3) <*> Just 9  
Just 12  
ghci> pure (+3) <*> Just 10  
Just 13  
ghci> pure (+3) <*> Just 9  
Just 12  
ghci> Just (++"hahah") <*> Nothing  
Nothing  
ghci> Nothing <*> Just "woot"  
Nothing  
We see how doing pure (+3) and Just (+3) is the same in this case. Use pure if you're dealing with Maybe values in an applicative context (i.e. using them with <*>), otherwise stick to Just. The first four input lines demonstrate how the function is extracted and then mapped, but in this case, they could have been achieved by just mapping unwrapped functions over functors. The last line is interesting, because we try to extract a function from a Nothing and then map it over something, which of course results in a Nothing.

With normal functors, you can just map a function over a functor and then you can't get the result out in any general way, even if the result is a partially applied function. Applicative functors, on the other hand, allow you to operate on several functors with a single function. Check out this piece of code:

ghci> pure (+) <*> Just 3 <*> Just 5  
Just 8  
ghci> pure (+) <*> Just 3 <*> Nothing  
Nothing  
ghci> pure (+) <*> Nothing <*> Just 5  
Nothing  
whaale
What's going on here? Let's take a look, step by step. <*> is left-associative, which means that pure (+) <*> Just 3 <*> Just 5 is the same as (pure (+) <*> Just 3) <*> Just 5. First, the + function is put in a functor, which is in this case a Maybe value that contains the function. So at first, we have pure (+), which is Just (+). Next, Just (+) <*> Just 3 happens. The result of this is Just (3+). This is because of partial application. Only applying 3 to the + function results in a function that takes one parameter and adds 3 to it. Finally, Just (3+) <*> Just 5 is carried out, which results in a Just 8.

Isn't this awesome?! Applicative functors and the applicative style of doing pure f <*> x <*> y <*> ... allow us to take a function that expects parameters that aren't necessarily wrapped in functors and use that function to operate on several values that are in functor contexts. The function can take as many parameters as we want, because it's always partially applied step by step between occurences of <*>.

This becomes even more handy and apparent if we consider the fact that pure f <*> x equals fmap f x. This is one of the applicative laws. We'll take a closer look at them later, but for now, we can sort of intuitively see that this is so. Think about it, it makes sense. Like we said before, pure puts a value in a default context. If we just put a function in a default context and then extract and apply it to a value inside another applicative functor, we did the same as just mapping that function over that applicative functor. Instead of writing pure f <*> x <*> y <*> ..., we can write fmap f x <*> y <*> .... This is why Control.Applicative exports a function called <$>, which is just fmap as an infix operator. Here's how it's defined:

(<$>) :: (Functor f) => (a -> b) -> f a -> f b  
f <$> x = fmap f x  
Yo! Quick reminder: type variables are independent of parameter names or other value names. The f in the function declaration here is a type variable with a class constraint saying that any type constructor that replaces f should be in the Functor typeclass. The f in the function body denotes a function that we map over x. The fact that we used f to represent both of those doesn't mean that they somehow represent the same thing.
By using <$>, the applicative style really shines, because now if we want to apply a function f between three applicative functors, we can write f <$> x <*> y <*> z. If the parameters weren't applicative functors but normal values, we'd write f x y z.

Let's take a closer look at how this works. We have a value of Just "johntra" and a value of Just "volta" and we want to join them into one String inside a Maybe functor. We do this:

ghci> (++) <$> Just "johntra" <*> Just "volta"  
Just "johntravolta"  
Before we see how this happens, compare the above line with this:

ghci> (++) "johntra" "volta"  
"johntravolta"  
Awesome! To use a normal function on applicative functors, just sprinkle some <$> and <*> about and the function will operate on applicatives and return an applicative. How cool is that?

Anyway, when we do (++) <$> Just "johntra" <*> Just "volta", first (++), which has a type of (++) :: [a] -> [a] -> [a] gets mapped over Just "johntra", resulting in a value that's the same as Just ("johntra"++) and has a type of Maybe ([Char] -> [Char]). Notice how the first parameter of (++) got eaten up and how the as turned into Chars. And now Just ("johntra"++) <*> Just "volta" happens, which takes the function out of the Just and maps it over Just "volta", resulting in Just "johntravolta". Had any of the two values been Nothing, the result would have also been Nothing.

So far, we've only used Maybe in our examples and you might be thinking that applicative functors are all about Maybe. There are loads of other instances of Applicative, so let's go and meet them!

Lists (actually the list type constructor, []) are applicative functors. What a suprise! Here's how [] is an instance of Applicative:

instance Applicative [] where  
    pure x = [x]  
    fs <*> xs = [f x | f <- fs, x <- xs]  
Earlier, we said that pure takes a value and puts it in a default context. Or in other words, a minimal context that still yields that value. The minimal context for lists would be the empty list, [], but the empty list represents the lack of a value, so it can't hold in itself the value that we used pure on. That's why pure takes a value and puts it in a singleton list. Similarly, the minimal context for the Maybe applicative functor would be a Nothing, but it represents the lack of a value instead of a value, so pure is implemented as Just in the instance implementation for Maybe.

ghci> pure "Hey" :: [String]  
["Hey"]  
ghci> pure "Hey" :: Maybe String  
Just "Hey"  
What about <*>? If we look at what <*>'s type would be if it were limited only to lists, we get (<*>) :: [a -> b] -> [a] -> [b]. It's implemented with a list comprehension. <*> has to somehow extract the function out of its left parameter and then map it over the right parameter. But the thing here is that the left list can have zero functions, one function, or several functions inside it. The right list can also hold several values. That's why we use a list comprehension to draw from both lists. We apply every possible function from the left list to every possible value from the right list. The resulting list has every possible combination of applying a function from the left list to a value in the right one.

ghci> [(*0),(+100),(^2)] <*> [1,2,3]  
[0,0,0,101,102,103,1,4,9]  
The left list has three functions and the right list has three values, so the resulting list will have nine elements. Every function in the left list is applied to every function in the right one. If we have a list of functions that take two parameters, we can apply those functions between two lists.

ghci> [(+),(*)] <*> [1,2] <*> [3,4]  
[4,5,5,6,3,4,6,8]  
Because <*> is left-associative, [(+),(*)] <*> [1,2] happens first, resulting in a list that's the same as [(1+),(2+),(1*),(2*)], because every function on the left gets applied to every value on the right. Then, [(1+),(2+),(1*),(2*)] <*> [3,4] happens, which produces the final result.

Using the applicative style with lists is fun! Watch:

ghci> (++) <$> ["ha","heh","hmm"] <*> ["?","!","."]  
["ha?","ha!","ha.","heh?","heh!","heh.","hmm?","hmm!","hmm."]  
Again, see how we used a normal function that takes two strings between two applicative functors of strings just by inserting the appropriate applicative operators.

You can view lists as non-deterministic computations. A value like 100 or "what" can be viewed as a deterministic computation that has only one result, whereas a list like [1,2,3] can be viewed as a computation that can't decide on which result it wants to have, so it presents us with all of the possible results. So when you do something like (+) <$> [1,2,3] <*> [4,5,6], you can think of it as adding together two non-deterministic computations with +, only to produce another non-deterministic computation that's even less sure about its result.

Using the applicative style on lists is often a good replacement for list comprehensions. In the second chapter, we wanted to see all the possible products of [2,5,10] and [8,10,11], so we did this:

ghci> [ x*y | x <- [2,5,10], y <- [8,10,11]]     
[16,20,22,40,50,55,80,100,110]     
We're just drawing from two lists and applying a function between every combination of elements. This can be done in the applicative style as well:

ghci> (*) <$> [2,5,10] <*> [8,10,11]  
[16,20,22,40,50,55,80,100,110]  
This seems clearer to me, because it's easier to see that we're just calling * between two non-deterministic computations. If we wanted all possible products of those two lists that are more than 50, we'd just do:

ghci> filter (>50) $ (*) <$> [2,5,10] <*> [8,10,11]  
[55,80,100,110]  
It's easy to see how pure f <*> xs equals fmap f xs with lists. pure f is just [f] and [f] <*> xs will apply every function in the left list to every value in the right one, but there's just one function in the left list, so it's like mapping.

Another instance of Applicative that we've already encountered is IO. This is how the instance is implemented:

instance Applicative IO where  
    pure = return  
    a <*> b = do  
        f <- a  
        x <- b  
        return (f x)  
ahahahah!
Since pure is all about putting a value in a minimal context that still holds it as its result, it makes sense that pure is just return, because return does exactly that; it makes an I/O action that doesn't do anything, it just yields some value as its result, but it doesn't really do any I/O operations like printing to the terminal or reading from a file.

If <*> were specialized for IO it would have a type of (<*>) :: IO (a -> b) -> IO a -> IO b. It would take an I/O action that yields a function as its result and another I/O action and create a new I/O action from those two that, when performed, first performs the first one to get the function and then performs the second one to get the value and then it would yield that function applied to the value as its result. We used do syntax to implement it here. Remember, do syntax is about taking several I/O actions and gluing them into one, which is exactly what we do here.

With Maybe and [], we could think of <*> as simply extracting a function from its left parameter and then sort of applying it over the right one. With IO, extracting is still in the game, but now we also have a notion of sequencing, because we're taking two I/O actions and we're sequencing, or gluing, them into one. We have to extract the function from the first I/O action, but to extract a result from an I/O action, it has to be performed.

Consider this:

myAction :: IO String  
myAction = do  
    a <- getLine  
    b <- getLine  
    return $ a ++ b  
This is an I/O action that will prompt the user for two lines and yield as its result those two lines concatenated. We achieved it by gluing together two getLine I/O actions and a return, because we wanted our new glued I/O action to hold the result of a ++ b. Another way of writing this would be to use the applicative style.

myAction :: IO String  
myAction = (++) <$> getLine <*> getLine  
What we were doing before was making an I/O action that applied a function between the results of two other I/O actions, and this is the same thing. Remember, getLine is an I/O action with the type getLine :: IO String. When we use <*> between two applicative functors, the result is an applicative functor, so this all makes sense.

If we regress to the box analogy, we can imagine getLine as a box that will go out into the real world and fetch us a string. Doing (++) <$> getLine <*> getLine makes a new, bigger box that sends those two boxes out to fetch lines from the terminal and then presents the concatenation of those two lines as its result.

The type of the expression (++) <$> getLine <*> getLine is IO String, which means that this expression is a completely normal I/O action like any other, which also holds a result value inside it, just like other I/O actions. That's why we can do stuff like:

main = do  
    a <- (++) <$> getLine <*> getLine  
    putStrLn $ "The two lines concatenated turn out to be: " ++ a  
If you ever find yourself binding some I/O actions to names and then calling some function on them and presenting that as the result by using return, consider using the applicative style because it's arguably a bit more concise and terse.

Another instance of Applicative is (->) r, so functions. They are rarely used with the applicative style outside of code golf, but they're still interesting as applicatives, so let's take a look at how the function instance is implemented.

If you're confused about what (->) r means, check out the previous section where we explain how (->) r is a functor.
instance Applicative ((->) r) where  
    pure x = (\_ -> x)  
    f <*> g = \x -> f x (g x)  
When we wrap a value into an applicative functor with pure, the result it yields always has to be that value. A minimal default context that still yields that value as a result. That's why in the function instance implementation, pure takes a value and creates a function that ignores its parameter and always returns that value. If we look at the type for pure, but specialized for the (->) r instance, it's pure :: a -> (r -> a).

ghci> (pure 3) "blah"  
3  
Because of currying, function application is left-associative, so we can omit the parentheses.

ghci> pure 3 "blah"  
3  
The instance implementation for <*> is a bit cryptic, so it's best if we just take a look at how to use functions as applicative functors in the applicative style.

ghci> :t (+) <$> (+3) <*> (*100)  
(+) <$> (+3) <*> (*100) :: (Num a) => a -> a  
ghci> (+) <$> (+3) <*> (*100) $ 5  
508  
Calling <*> with two applicative functors results in an applicative functor, so if we use it on two functions, we get back a function. So what goes on here? When we do (+) <$> (+3) <*> (*100), we're making a function that will use + on the results of (+3) and (*100) and return that. To demonstrate on a real example, when we did (+) <$> (+3) <*> (*100) $ 5, the 5 first got applied to (+3) and (*100), resulting in 8 and 500. Then, + gets called with 8 and 500, resulting in 508.

ghci> (\x y z -> [x,y,z]) <$> (+3) <*> (*2) <*> (/2) $ 5  
[8.0,10.0,2.5]  
SLAP
Same here. We create a function that will call the function \x y z -> [x,y,z] with the eventual results from (+3), (*2) and (/2). The 5 gets fed to each of the three functions and then \x y z -> [x, y, z] gets called with those results.

You can think of functions as boxes that contain their eventual results, so doing k <$> f <*> g creates a function that will call k with the eventual results from f and g. When we do something like (+) <$> Just 3 <*> Just 5, we're using + on values that might or might not be there, which also results in a value that might or might not be there. When we do (+) <$> (+10) <*> (+5), we're using + on the future return values of (+10) and (+5) and the result is also something that will produce a value only when called with a parameter.

We don't often use functions as applicatives, but this is still really interesting. It's not very important that you get how the (->) r instance for Applicative works, so don't despair if you're not getting this right now. Try playing with the applicative style and functions to build up an intuition for functions as applicatives.

An instance of Applicative that we haven't encountered yet is ZipList, and it lives in Control.Applicative.

It turns out there are actually more ways for lists to be applicative functors. One way is the one we already covered, which says that calling <*> with a list of functions and a list of values results in a list which has all the possible combinations of applying functions from the left list to the values in the right list. If we do [(+3),(*2)] <*> [1,2], (+3) will be applied to both 1 and 2 and (*2) will also be applied to both 1 and 2, resulting in a list that has four elements, namely [4,5,2,4].

However, [(+3),(*2)] <*> [1,2] could also work in such a way that the first function in the left list gets applied to the first value in the right one, the second function gets applied to the second value, and so on. That would result in a list with two values, namely [4,4]. You could look at it as [1 + 3, 2 * 2].

Because one type can't have two instances for the same typeclass, the ZipList a type was introduced, which has one constructor ZipList that has just one field, and that field is a list. Here's the instance:

instance Applicative ZipList where  
        pure x = ZipList (repeat x)  
        ZipList fs <*> ZipList xs = ZipList (zipWith (\f x -> f x) fs xs)  
<*> does just what we said. It applies the first function to the first value, the second function to the second value, etc. This is done with zipWith (\f x -> f x) fs xs. Because of how zipWith works, the resulting list will be as long as the shorter of the two lists.

pure is also interesting here. It takes a value and puts it in a list that just has that value repeating indefinitely. pure "haha" results in ZipList (["haha","haha","haha".... This might be a bit confusing since we said that pure should put a value in a minimal context that still yields that value. And you might be thinking that an infinite list of something is hardly minimal. But it makes sense with zip lists, because it has to produce the value on every position. This also satisfies the law that pure f <*> xs should equal fmap f xs. If pure 3 just returned ZipList [3], pure (*2) <*> ZipList [1,5,10] would result in ZipList [2], because the resulting list of two zipped lists has the length of the shorter of the two. If we zip a finite list with an infinite list, the length of the resulting list will always be equal to the length of the finite list.

So how do zip lists work in an applicative style? Let's see. Oh, the ZipList a type doesn't have a Show instance, so we have to use the getZipList function to extract a raw list out of a zip list.

ghci> getZipList $ (+) <$> ZipList [1,2,3] <*> ZipList [100,100,100]  
[101,102,103]  
ghci> getZipList $ (+) <$> ZipList [1,2,3] <*> ZipList [100,100..]  
[101,102,103]  
ghci> getZipList $ max <$> ZipList [1,2,3,4,5,3] <*> ZipList [5,3,1,2]  
[5,3,3,4]  
ghci> getZipList $ (,,) <$> ZipList "dog" <*> ZipList "cat" <*> ZipList "rat"  
[('d','c','r'),('o','a','a'),('g','t','t')]  
The (,,) function is the same as \x y z -> (x,y,z). Also, the (,) function is the same as \x y -> (x,y).
Aside from zipWith, the standard library has functions such as zipWith3, zipWith4, all the way up to 7. zipWith takes a function that takes two parameters and zips two lists with it. zipWith3 takes a function that takes three parameters and zips three lists with it, and so on. By using zip lists with an applicative style, we don't have to have a separate zip function for each number of lists that we want to zip together. We just use the applicative style to zip together an arbitrary amount of lists with a function, and that's pretty cool.

Control.Applicative defines a function that's called liftA2, which has a type of liftA2 :: (Applicative f) => (a -> b -> c) -> f a -> f b -> f c . It's defined like this:

liftA2 :: (Applicative f) => (a -> b -> c) -> f a -> f b -> f c  
liftA2 f a b = f <$> a <*> b  
Nothing special, it just applies a function between two applicatives, hiding the applicative style that we've become familiar with. The reason we're looking at it is because it clearly showcases why applicative functors are more powerful than just ordinary functors. With ordinary functors, we can just map functions over one functor. But with applicative functors, we can apply a function between several functors. It's also interesting to look at this function's type as (a -> b -> c) -> (f a -> f b -> f c). When we look at it like this, we can say that liftA2 takes a normal binary function and promotes it to a function that operates on two functors.

Here's an interesting concept: we can take two applicative functors and combine them into one applicative functor that has inside it the results of those two applicative functors in a list. For instance, we have Just 3 and Just 4. Let's assume that the second one has a singleton list inside it, because that's really easy to achieve:

ghci> fmap (\x -> [x]) (Just 4)  
Just [4]  
OK, so let's say we have Just 3 and Just [4]. How do we get Just [3,4]? Easy.

ghci> liftA2 (:) (Just 3) (Just [4])  
Just [3,4]  
ghci> (:) <$> Just 3 <*> Just [4]  
Just [3,4]  
Remember, : is a function that takes an element and a list and returns a new list with that element at the beginning. Now that we have Just [3,4], could we combine that with Just 2 to produce Just [2,3,4]? Of course we could. It seems that we can combine any amount of applicatives into one applicative that has a list of the results of those applicatives inside it. Let's try implementing a function that takes a list of applicatives and returns an applicative that has a list as its result value. We'll call it sequenceA.

sequenceA :: (Applicative f) => [f a] -> f [a]  
sequenceA [] = pure []  
sequenceA (x:xs) = (:) <$> x <*> sequenceA xs  
Ah, recursion! First, we look at the type. It will transform a list of applicatives into an applicative with a list. From that, we can lay some groundwork for an edge condition. If we want to turn an empty list into an applicative with a list of results, well, we just put an empty list in a default context. Now comes the recursion. If we have a list with a head and a tail (remember, x is an applicative and xs is a list of them), we call sequenceA on the tail, which results in an applicative with a list. Then, we just prepend the value inside the applicative x into that applicative with a list, and that's it!

So if we do sequenceA [Just 1, Just 2], that's (:) <$> Just 1 <*> sequenceA [Just 2] . That equals (:) <$> Just 1 <*> ((:) <$> Just 2 <*> sequenceA []). Ah! We know that sequenceA [] ends up as being Just [], so this expression is now (:) <$> Just 1 <*> ((:) <$> Just 2 <*> Just []), which is (:) <$> Just 1 <*> Just [2], which is Just [1,2]!

Another way to implement sequenceA is with a fold. Remember, pretty much any function where we go over a list element by element and accumulate a result along the way can be implemented with a fold.

sequenceA :: (Applicative f) => [f a] -> f [a]  
sequenceA = foldr (liftA2 (:)) (pure [])  
We approach the list from the right and start off with an accumulator value of pure []. We do liftA2 (:) between the accumulator and the last element of the list, which results in an applicative that has a singleton in it. Then we do liftA2 (:) with the now last element and the current accumulator and so on, until we're left with just the accumulator, which holds a list of the results of all the applicatives.

Let's give our function a whirl on some applicatives.

ghci> sequenceA [Just 3, Just 2, Just 1]  
Just [3,2,1]  
ghci> sequenceA [Just 3, Nothing, Just 1]  
Nothing  
ghci> sequenceA [(+3),(+2),(+1)] 3  
[6,5,4]  
ghci> sequenceA [[1,2,3],[4,5,6]]  
[[1,4],[1,5],[1,6],[2,4],[2,5],[2,6],[3,4],[3,5],[3,6]]  
ghci> sequenceA [[1,2,3],[4,5,6],[3,4,4],[]]  
[]  
Ah! Pretty cool. When used on Maybe values, sequenceA creates a Maybe value with all the results inside it as a list. If one of the values was Nothing, then the result is also a Nothing. This is cool when you have a list of Maybe values and you're interested in the values only if none of them is a Nothing.

When used with functions, sequenceA takes a list of functions and returns a function that returns a list. In our example, we made a function that took a number as a parameter and applied it to each function in the list and then returned a list of results. sequenceA [(+3),(+2),(+1)] 3 will call (+3) with 3, (+2) with 3 and (+1) with 3 and present all those results as a list.

Doing (+) <$> (+3) <*> (*2) will create a function that takes a parameter, feeds it to both (+3) and (*2) and then calls + with those two results. In the same vein, it makes sense that sequenceA [(+3),(*2)] makes a function that takes a parameter and feeds it to all of the functions in the list. Instead of calling + with the results of the functions, a combination of : and pure [] is used to gather those results in a list, which is the result of that function.

Using sequenceA is cool when we have a list of functions and we want to feed the same input to all of them and then view the list of results. For instance, we have a number and we're wondering whether it satisfies all of the predicates in a list. One way to do that would be like so:

ghci> map (\f -> f 7) [(>4),(<10),odd]  
[True,True,True]  
ghci> and $ map (\f -> f 7) [(>4),(<10),odd]  
True  
Remember, and takes a list of booleans and returns True if they're all True. Another way to achieve the same thing would be with sequenceA:

ghci> sequenceA [(>4),(<10),odd] 7  
[True,True,True]  
ghci> and $ sequenceA [(>4),(<10),odd] 7  
True  
sequenceA [(>4),(<10),odd] creates a function that will take a number and feed it to all of the predicates in [(>4),(<10),odd] and return a list of booleans. It turns a list with the type (Num a) => [a -> Bool] into a function with the type (Num a) => a -> [Bool]. Pretty neat, huh?

Because lists are homogenous, all the functions in the list have to be functions of the same type, of course. You can't have a list like [ord, (+3)], because ord takes a character and returns a number, whereas (+3) takes a number and returns a number.

When used with [], sequenceA takes a list of lists and returns a list of lists. Hmm, interesting. It actually creates lists that have all possible combinations of their elements. For illustration, here's the above done with sequenceA and then done with a list comprehension:

ghci> sequenceA [[1,2,3],[4,5,6]]  
[[1,4],[1,5],[1,6],[2,4],[2,5],[2,6],[3,4],[3,5],[3,6]]  
ghci> [[x,y] | x <- [1,2,3], y <- [4,5,6]]  
[[1,4],[1,5],[1,6],[2,4],[2,5],[2,6],[3,4],[3,5],[3,6]]  
ghci> sequenceA [[1,2],[3,4]]  
[[1,3],[1,4],[2,3],[2,4]]  
ghci> [[x,y] | x <- [1,2], y <- [3,4]]  
[[1,3],[1,4],[2,3],[2,4]]  
ghci> sequenceA [[1,2],[3,4],[5,6]]  
[[1,3,5],[1,3,6],[1,4,5],[1,4,6],[2,3,5],[2,3,6],[2,4,5],[2,4,6]]  
ghci> [[x,y,z] | x <- [1,2], y <- [3,4], z <- [5,6]]  
[[1,3,5],[1,3,6],[1,4,5],[1,4,6],[2,3,5],[2,3,6],[2,4,5],[2,4,6]]  
This might be a bit hard to grasp, but if you play with it for a while, you'll see how it works. Let's say that we're doing sequenceA [[1,2],[3,4]]. To see how this happens, let's use the sequenceA (x:xs) = (:) <$> x <*> sequenceA xs definition of sequenceA and the edge condition sequenceA [] = pure []. You don't have to follow this evaluation, but it might help you if have trouble imagining how sequenceA works on lists of lists, because it can be a bit mind-bending.

We start off with sequenceA [[1,2],[3,4]]
That evaluates to (:) <$> [1,2] <*> sequenceA [[3,4]]
Evaluating the inner sequenceA further, we get (:) <$> [1,2] <*> ((:) <$> [3,4] <*> sequenceA [])
We've reached the edge condition, so this is now (:) <$> [1,2] <*> ((:) <$> [3,4] <*> [[]])
Now, we evaluate the (:) <$> [3,4] <*> [[]] part, which will use : with every possible value in the left list (possible values are 3 and 4) with every possible value on the right list (only possible value is []), which results in [3:[], 4:[]], which is [[3],[4]]. So now we have (:) <$> [1,2] <*> [[3],[4]]
Now, : is used with every possible value from the left list (1 and 2) with every possible value in the right list ([3] and [4]), which results in [1:[3], 1:[4], 2:[3], 2:[4]], which is [[1,3],[1,4],[2,3],[2,4]
Doing (+) <$> [1,2] <*> [4,5,6]results in a non-deterministic computation x + y where x takes on every value from [1,2] and y takes on every value from [4,5,6]. We represent that as a list which holds all of the possible results. Similarly, when we do sequence [[1,2],[3,4],[5,6],[7,8]], the result is a non-deterministic computation [x,y,z,w], where x takes on every value from [1,2], y takes on every value from [3,4] and so on. To represent the result of that non-deterministic computation, we use a list, where each element in the list is one possible list. That's why the result is a list of lists.

When used with I/O actions, sequenceA is the same thing as sequence! It takes a list of I/O actions and returns an I/O action that will perform each of those actions and have as its result a list of the results of those I/O actions. That's because to turn an [IO a] value into an IO [a] value, to make an I/O action that yields a list of results when performed, all those I/O actions have to be sequenced so that they're then performed one after the other when evaluation is forced. You can't get the result of an I/O action without performing it.

ghci> sequenceA [getLine, getLine, getLine]  
heyh  
ho  
woo  
["heyh","ho","woo"]  
Like normal functors, applicative functors come with a few laws. The most important one is the one that we already mentioned, namely that pure f <*> x = fmap f x holds. As an exercise, you can prove this law for some of the applicative functors that we've met in this chapter.The other functor laws are:

pure id <*> v = v
pure (.) <*> u <*> v <*> w = u <*> (v <*> w)
pure f <*> pure x = pure (f x)
u <*> pure y = pure ($ y) <*> u
We won't go over them in detail right now because that would take up a lot of pages and it would probably be kind of boring, but if you're up to the task, you can take a closer look at them and see if they hold for some of the instances.

In conclusion, applicative functors aren't just interesting, they're also useful, because they allow us to combine different computations, such as I/O computations, non-deterministic computations, computations that might have failed, etc. by using the applicative style. Just by using <$> and <*> we can use normal functions to uniformly operate on any number of applicative functors and take advantage of the semantics of each one.
