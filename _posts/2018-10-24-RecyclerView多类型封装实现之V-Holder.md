---
title: Android RecyclerView多类型封装实现之V-Holder
categories:
 - android
tags:
---

最近恰好项目中用到的RV多种类型,然后把这块做了一些简要的封装，市面上也有很多这方面的实现，感觉有些用着不是很爽，于是就开始造轮子，虽然说不提倡造轮子。

 进入正题咋们先看看RecyclerView多类型处理传统的写法：
   首先定义三种常量  表示三种item类型

      public static final int TYPE_1 = 0;
      public static final int TYPE_2 = 1;
      public static final int TYPE_3 = 2;



  然后重写getItemViewType方法 根据条件返回对应的item的类型

    @Override
    public final int getItemViewType(int position) {
        TypeBean mTypeBean = mData.get(position)
        if(mTypeBean == 0){
           return TYPE_1;
        }else if(mTypeBean == 01){
           return TYPE_2;
        }else{
           return TYPE_3;
        }
    }


  上面定义不同类型常量之后并且getItemViewType返回了对应的类型常量，于是在onCreateViewHolder里根据viewtype来判断 所引用的item布局类型并且创建不同布局的holder。

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

        View view;
       //根据viewType来判断不同的布局并且创建其ViewHolder
       if (viewType == TYPE_PULL_IMAGE) {
         view =View.inflate(parent.getContext(),R.layout.item_1,null);
         return new ItemHolder1(view);
       } else if (viewType == TYPE_RIGHT_IMAGE) {
         view =View.inflate(parent.getContext(),R.layout.item_2,null);
         return new ItemHolder2(view);
       } else {
         view =View.inflate(parent.getContext(),R.layout.item_3,null);
         return new ItemHolder3(view);
       }
    }



在onBindViewHolder方法中绑定数据。

     @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {

    }


 创建对应的ViewHolder

      //ItemHolder1
      static class ItemHolder1 extends RecyclerView.ViewHolder {

            public ItemHolder1(View view) {
                     super(view);
                     //初始化控件
                     ...

            }
      }

      //ItemHolder2
       static class ItemHolder2 extends RecyclerView.ViewHolder {

            public ItemHolder2(View view) {
                     super(view);
                     //初始化控件
                     ...

            }
      }

      //ItemHolder3
       static class ItemHolder3 extends RecyclerView.ViewHolder {

            public ItemHolder3(View view) {
                     super(view);
                     //初始化控件
                     ...

            }
      }


 这样传统的多类型RecyclerView大致可以实现了，但是如果每一个页面都诸如此类的写下去，相对来说非常繁琐，并且代码冗余量也非常大，同时复用性也非常差，于是乎，便对其提取，封装,首先我们肯定是要对getItemViewType()方法进行处理，避免定义大量类型的常量，此为问题一。其次就是不同类型的ViewHolder创建，抽取onCreateViewHolder中代码,此为问题二。先把ViewHolde抽象：

    public abstract class VHolder<T, VH extends ViewHolder> {

      protected abstract @NonNull
      VH onCreateViewHolder(@NonNull LayoutInflater inflater, @NonNull ViewGroup parent);


      protected abstract void onBindViewHolder(@NonNull VH holder, @NonNull T item);


      protected void onBindViewHolder(@NonNull VH holder, @NonNull T item, @NonNull List<Object> payloads) {
        onBindViewHolder(holder, item);
     }
    }
VHolder中泛型参数是数据Bean和ViewHolder子类，定义了2个抽象方法onCreateViewHolder和onBindViewHolder延用了官方的形式，因此我们的ViewHolder需继承VHolder，对VHolder进一步抽取封装


    public abstract class AbsItemHolder<T, VH extends AbsHolder> extends VHolder<T, VH> {

    protected Context mContext;

    public AbsItemHolder(Context context) {
        this.mContext = context;
    }

    @Override
    protected @NonNull
    VH onCreateViewHolder(@NonNull LayoutInflater inflater, @NonNull ViewGroup parent) {
        return createViewHolder(inflater.inflate(getLayoutResId(), parent, false));
    }

    public abstract int getLayoutResId();


    public abstract VH createViewHolder(View view);

    }



    public abstract class AbsHolder extends RecyclerView.ViewHolder {

    private final SparseArray<View> views;

    public View convertView;


    public AbsHolder(final View view) {
        super(view);
        this.views = new SparseArray<>();
        this.convertView = view;
    }

    public <T extends View> T getViewById(@IdRes int viewId) {
        View view = views.get(viewId);
        if (view == null) {
            view = itemView.findViewById(viewId);
            views.put(viewId, view);
        }
        return (T) view;
    }
    }

至此我们的AbsItemHolder基类编写完成。


 一般来说通常情况下，客户端展示的类型基本数据都是和服务端提前约定好的返回什么样数据结构。因此客户端要展示什么样的UI基本都是定死的，那么也就是说假如约定的有8中类型的UI,那么客户端展示的UI类型就是小于或等于8种类型。那么我们期望对于这种多类型的item是一个可配置的，接下来我们通过Builder模式来配置。写一个DelegateAdapter继承自系统的RecyclerView.Adapter，在其内部写一个静态的Builder类，

    public class DelegateAdapter extends RecyclerView.Adapter<ViewHolder> {
      private  List<Class<?>> classes;
      private List<VHolder<?, ?>> vHolders

      public DelegateAdapter(Builder builder) {
        this.classes = builder.classes;
        this.vHolders = builder.vHolders;
      }


     public static class Builder<T> {

      private final
      List<Class<?>> classes;
      private final
      List<VHolder<?, ?>> vHolders;

       public Builder() {
           this.classes = new ArrayList<>();
           this.vHolders = new ArrayList<>();
       }

       public Builder bind(Class<? extends T> clazz, VHolder itemView) {
             classes.add(clazz);
             vHolders.add(binder);
            return this;
       }

        public DelegateAdapter build() {
            return new DelegateAdapter(this);
        }
      }
    }



 我们希望结果是这样，干净利落。

    DelegateAdapter  adapter = new DelegateAdapter.Builder()
                .bind(Item1.class, new ItemHolder1(this))
                .bind(Item2.class, new ItemHolder2(this))
                .bind(Item3.class, new ItemHolder3(this))
                .build();

到这里我们发现对应的不同类型的Item.class和ItemHolder都是是有序的存储在list里面，那么我们是不是可以通过获取list的索引作为我们的getItemViewType的类型表示呢，答案肯定是No problem，


      @Override
    public final int getItemViewType(int position) {
       //获取position位置对应的Item.class
       Object itemBean=datas.get(position);
       //找出Item.class在classes中储存的index
       int index = classes.indexOf(itemBean.getClass());
       return index;
    }

获取到了item类型的标识，那么接下来我们可以创建对应的ViewHolder

    @Override
    public final ViewHolder onCreateViewHolder(ViewGroup parent, int viewType)    {
        if (null == inflater) {
            inflater = LayoutInflater.from(parent.getContext());
        }
       //通过viewType获取vHolders中VHolder;
        VHolder<?, ?> vHolder = vHolders.getItemView(viewType);
        //上文中定义了抽象类VHolder，这里就通过vHolder分发调用onCreateViewHolder方法
        return vHolder.onCreateViewHolder(inflater, parent);
    }


  同理onBindViewHolder也是通过vHolder分发调用onBindViewHolder方法

     @Override
    public final void onBindViewHolder(ViewHolder holder, final int position, @NonNull List<Object> payloads) {
        VHolder vHolder = vHolders.getItemView(holder.getItemViewType());
        vHolder.onBindViewHolder(holder, datas.get(position), payloads);

    }



 貌似感觉封装的差不多，接下来我们再改造原始的实现方式：

 首先定义adapter，没毛病。

    DelegateAdapter  adapter = new DelegateAdapter.Builder()
            .bind(Item1.class, new ItemHolder1(this))
            .bind(Item2.class, new ItemHolder2(this))
            .bind(Item3.class, new ItemHolder3(this))
            .build();

  然后编写对应的ViewHolder，如ItemHolder1是这样的：

    public class ItemHolder1 extends AbsItemHolder<Item1, ItemHolder1.ViewHolder> {
        public ItemHolder1(Context context) {
          super(context);
        }
        // item的布局
        @Override
        public int getLayoutResId() {
            return R.layout.item_1;
        }
        @Override
        public ViewHolder createViewHolder(View view) {
           return new ViewHolder(view);
        }
        @Override
        protected void onBindViewHolder(@NonNull ViewHolder holder, @NonNull Item1 item) {
           //绑定数据
       }
       static class ViewHolder extends AbsHolder {

           ViewHolder(@NonNull final View itemView) {
               super(itemView);
           }

       }

    }

 ItemHolder2是这样的：

     public class ItemHolder2 extends AbsItemHolder<Item2, ItemHolder2.ViewHolder> {
        public ItemHolder2(Context context) {
          super(context);
        }
        // item的布局
        @Override
        public int getLayoutResId() {
            return R.layout.item_2;
        }
        @Override
        public ViewHolder createViewHolder(View view) {
           return new ViewHolder(view);
        }
        @Override
        protected void onBindViewHolder(@NonNull ViewHolder holder, @NonNull Item2 item) {
           //绑定数据
       }
       static class ViewHolder extends AbsHolder {

           ViewHolder(@NonNull final View itemView) {
               super(itemView);
           }

       }

    }

 ItemHolder3是这样的：

     public class ItemHolder3 extends AbsItemHolder<Item3, ItemHolder3.ViewHolder> {
        public ItemHolder3(Context context) {
          super(context);
        }
        // item的布局
        @Override
        public int getLayoutResId() {
            return R.layout.item_3;
        }
        @Override
        public ViewHolder createViewHolder(View view) {
           return new ViewHolder(view);
        }
        @Override
        protected void onBindViewHolder(@NonNull ViewHolder holder, @NonNull Item3 item) {
           //绑定数据
       }
       static class ViewHolder extends AbsHolder {

           ViewHolder(@NonNull final View itemView) {
               super(itemView);
           }

       }

    }

 把每一种类型都拆分出来，提高复用性，后续只需要编写对应的ViewHolder就行。但是在实际开发中我们除了数据模型一对一以外还会出现一种模型对应多种类型，例如：

       {
    	"data":[
    			{
    				"title":"双11主会场",
    				"topicDesc":"超级红包",
    				"imgUrl":"",
    				"Type":1
    			},
    			{
    				"title":"双11主会场",
    				"topicDesc":"超级红包",
    				"imgUrl":"",
    				"Type":2
    			},
    			{
    				"title":"双11主会场",
    				"topicDesc":"超级红包",
    				"imgUrl":"",
    				"Type":3
    			},
    			]
     }

以上这个数据模型就是一种模型对应多种类型，根据每个模型的Type的取值来展示不同的UI，因此在前面的adapter中继续改造，

        /**
         * 数据类型一对多
         *
         * @param clazz
         * @param vHolders
         * @return
         */
        public Builder bindArray(Class<? extends T> clazz, VHolder... vHolders) {
            this.clazz = clazz;
            this.vHolders = vHolders;
            return this;
        }

  由于是一对多，所以需要bean并且根据不同的type返回不同viewholder,因此借助接口回调处理：

     定义Chain接口获取vHolders中的索引
     public interface Chain<T> {

          int indexItem(int var1, @NonNull T var2);
     }

     定义OneToMany接口返回当前模型中type对应的viewholder
     public interface OneToMany<T> {
        /**
         * 链接一对多
         * @param position
         * @param t
         * @return
         */
        Class<? extends VHolder<T, ?>> onItemView(int position, T t);
    }



    通过ChainOneToMany实现Chain接口获取到当前模型中type对应的viewholder在vHolders中的索引
    class ChainOneToMany<T> implements Chain<T> {

      private final OneToMany<T> oneToMany;

      private final VHolder<T, ?>[] vHolders;


      public ChainOneToMany(
            OneToMany<T> oneToMany,
            VHolder<T, ?>[] vHolders) {
          this.oneToMany = oneToMany;
          this.vHolders = vHolders;
      }

      @Override
      public int indexItem(int position, T t) {
         Class<?> aClass = oneToMany.onItemView(position, t);
          for (int i = 0; i < vHolders.length; i++) {
              if (vHolders[i].getClass().equals(aClass)) {
                    return i;
              }
          }
          return -1;

      }
     }

  因此对之前的Builder进行改进

     public static class Builder<T> {

      private final
      List<Class<?>> classes;
      private final
      List<VHolder<?, ?>> vHolders;
      private final
      List<Chain<?>> chains;

       public Builder() {
           this.classes = new ArrayList<>();
           this.vHolders = new ArrayList<>();
           this.chains = new ArrayList<>();
       }

       public Builder bind(Class<? extends T> clazz, VHolder itemView) {
             classes.add(clazz);
             vHolders.add(binder);
             chains.add(new DefaultChain<>());
            return this;
       }

       public Builder withClass(OneToMany<T> oneToMany) {
            chain<T> chain = new ChainOneToMany(oneToMany, vHolders);
            for (VHolder itemView : vHolders) {
                classes.add(clazz);
                vHolders.add(itemView);
                chains.add(chain);
            }
            return this;
        }

       public DelegateAdapter build() {
            return new DelegateAdapter(this);
        }
     }

 改进后的的结果：


    DelegateAdapter  adapter = new DelegateAdapter.Builder()
          //一对一
          .bind(Item1.class, new ItemHolder1(this))
          .bind(Item2.class, new ItemHolder2(this))
          .bind(Item3.class, new ItemHolder3(this))
          //一对多
          .bindArray(ItemData.class, new ItemHolder4(this), new ItemType5(this),new ItemHolder6(this))
          .withClass(new OneToMany<ItemData>() {
                  @Override
                  public Class<? extends VHolder<ItemData, ?>> onItemView(int position, ItemData itemData) {
                      if (itemData.type==1) {
                          return ItemHolder4.class;
                      }else if (itemData.type==2) {
                          return ItemHolder5.class;
                      }else if (itemData.type==3) {
                          return ItemHolder6.class;
                      }
                      return ItemHolder4.class;
                  }
              })
          .build();

基本上实现数据一对一和一对多的形式，如果单一的实现这种多类型的功能，作用是很大，大部分的页面都具有刷新或者加载更多的功能，于是乎，整合成了多类型的的刷新库github地址:<https://github.com/SelfZhangTQ/TRecyclerView> 小码自己已经商用，欢迎star，感谢支持

#### 实战使用项目 <br/>
github地址:<https://github.com/SelfZhangTQ/T-MVVM> 欢迎star，感谢支持<br/>




























