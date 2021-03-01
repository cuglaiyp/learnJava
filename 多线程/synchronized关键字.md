# synchronized关键字

synchornized可以修饰代码块、普通方法、静态方法。当synchornize(Object obj)修饰代码块或者普通方法时，每个对象都拥有一把锁，当synchornized(Object.class)或者修饰静态方法时，锁住的不是单个对象，而是这个类的所有对象。

~~~java
public class MyThread {

    public static void main(String[] args) throws InterruptedException {

        Task task1 = new Task();
        Task task2 = new Task();
        Thread t1 = new Thread(task1, "thread1");
        Thread t2 = new Thread(task2, "thread2");
        t1.start();
        t2.start();

    }


}

class Task implements Runnable {

    static int ticketCount = 10;

    public void run() {
        saleTicket();
    }

    private void saleTicket() {
        synchronized (this) {
            for (int i = 5; i > 0; i--) {
                System.out.println(Thread.currentThread().getName() + "卖出：第" + ticketCount--);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }

        }
    }

    private void readTicket() {
        System.out.println(ticketCount);
    }
}
~~~

---

结合这篇[synchronized.md](synchronized.md)一起看