# Layout & Boilerplate
First we should make a layout for our mutex. The minimum would be a Boolean to store if the mutex is locked or not and an Unsafe Cell to store the actual value inside the mutex. For now, we can use the following layout:

    use core::cell::UnsafeCell;
    use core::sync::atomic::{AtomicBool,Ordering};
    
    struct Mutex<T>{
        is_locked:AtomicBool,
        value:UnsafeCell,
    }

We also should fill out some boilerplate:

    impl<T> Mutex<T>{
        pub fn new(value:T)->Self{
            Self{is_locked:AtomicBool::new(false),UnsafeCell::new(value)}
        }
         
        pub fn get_mut(&mut self)->&mut T{
            self.value.get_mut()
        }
         
        pub fn into_inner(self)->T{
            self.value.into_inner()
        }
    }
And a sync impl:

    impl<T:Send+Sync> Sync for Mutex<T>{}

> Written with [StackEdit](https://stackedit.io/).
