# Adding RAII Support
While the version we made in the last chapter is usable, it requires unsafe code on the part of the user to unlock it and doesn't the user from using the mutable reference after the lock has been freed. We can solve these problems by using a RAII guard.

The guard itself will use the `core::ops::Deref` trait to allow it to still be used as if it were a mutable reference and the `Drop` trait to ensure that the mutex is unlocked when it leaves scope.

First we implement `Deref` and `DerefMut`:

    use core::ops::{Deref,MutDeref};
    
    struct MutexGuard<'a,T>(&'a Mutex<T>);
    
    impl<'a,T> Deref for MutexGuard<'a,T>{
        type Target = T;
        fn deref(&self)->&T{
            unsafe{&*self.0.value.get()}
        }
    }
     
    impl<'a,T> DerefMut for MutexGuard<'a,T>{
    

Then add a `Drop` implementation, to unlock the mutex when it leaves scope:

    impl<'a,T> Drop for MutexGuard<'a,T>{
        fn drop(&mut self){
            unsafe{self.0.unlock()}
        }
    }

Finally, we need to change our `lock()` method to return a mutex guard instead of a raw mutable reference (this should replace the existing lock method on mutex):

    impl<T> Mutex<T>{
        pub fn lock(&self)->MutexGuard<T>{
            while self.is_locked.compare_exchange(
                false,//expecting it to not be locked
                true ,//lock it if it is not already locked
                Ordering::Acquire,//If we succeed, use acquire ordering to prevent 
                Ordering::Release//If we faill to acquire the lock, it does not matter which ordering we choose, so choose the fastest.
            ).is_err(){}//If we fail to aquire the lock immediately try again.
            
            fence(Ordering::Acquire);//prevent writes to the data protected by the mutex 
            //from being reordered before the mutex is actually acquired.
            return MutexGuard(self);
        }
    }

> Written with [StackEdit](https://stackedit.io/).
