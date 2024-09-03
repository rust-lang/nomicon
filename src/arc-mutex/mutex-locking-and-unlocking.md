

# Locking and Unlocking
Locking and unlocking is simple with our layout; Make sure the mutex is in the correct state, then toggle it.

    use core::sync::fence;
     
    impl<T> Mutex<T>{
        pub fn lock(&self)->MutexGuard<T>{
            while self.is_locked.compare_exchange(
                false,//expecting it to not be locked
                true ,//lock it if it is not already locked
                Ordering::Acquire,//If we succeed, use acquire ordering to prevent 
                Ordering::Release//If we faill to acquire the lock, it does not matter which ordering we choose, so choose the fastest.
            ).is_err(){}//If we fail to aquire the lock immediately try again.
            
            fence(Ordering::Acquire);//prevent writes to the data protected by the mutex from being reordered before the mutex is actually acquired.
            //safety:The previous steps we took means that no one else has the lock
            unsafe{return &mut self.value.get()}
        }
        
        ///Safety:Make sure the caller has acquired the lock, and will not dereference the return mutable reference again.
        pub unsafe fn unlock(&self){
            if self.is_locked.compare_exchange(
                true,//The caller should ensure the mutex is locked
                false,//Mark the lock as unlocked
                Ordering::Release,
                Ordering::Relaxed,//We are about to panic anyway, the ordering doesn't matter
            ).is_err{
                panic!("Unlock called while the lock was not locked")
            };
            
            fence(Ordering::Acquire);//prevent writes to the data protected by the mutex 
            //from being reordered before the mutex is actually acquired.
        }
    }

 

> Written with [StackEdit](https://stackedit.io/).
