# O que é o DDD-Reflaction

DDD reflection tem o propósito de facilitar que seu dominio se torne idependente das entidades do hibernate.
Para isso foi desenvolvido dois utilitários que permite mapear seu modelo com as entidades do hibernate.

Exemplo:
  Hibernate entities

  ```javascript
   public class BusinessProposal {
   	    private Long id
   	    private User user;
   	    private List<ProposalSaleableItem> items;
   	    private ProposalTemperature temperature;

   	    //getters and setters
   }

   public class ProposalSaleableItem {
   		private Long id;
		private BigDecimal price;
		private Product product;
		private Integer quantity;

		//getters and setters
   }


   public class User {
   		private Long id;
   		private String name;
   		private List<Document> documents;

   		//getters and setters
   }

   public class Document {
   		public enum DocumentTypeEnum {PASSPORT, SOCIAL_SECURITY_CARD}
   		private String document;
  		private DocumentTypeEnum type;

  		//getters and setters
   }

   public class Product {
       private Long id;
       private String name;
   }
   ```

  Essas entidades respeitam o banco de dados e eu preciso que meu dominio fale de negócio e não de banco,
  também não posso deixar que uma alteração em uma tabela ou conjuntos de tabelas alterem a forma como meu
  negócio fala, então de forma nenhuma é uma boa prática usar entidades do hibernate como parte do seu dominio.

  Criando minhas entidades de dominio com os mapeamentos para a entidade de Banco de dados.

  ```javascript
  @EntityReference(BusinessProposal.class)
  Negotiation {

    private Long id;

    @EntityReference(User.class, fieldName="user")
    private Seller seller;

    @EntityReference(ProposalSaleableItem.class, fieldName="items")
    private List<SaleableNegociated> itemsNegotiated;

    private Status status;

    //getters and setters
  }

  @EntityReference(User.class)
  Seller {
    private Long id;
    private String name;

    //getters and setters
  }

  @EntityReference(ProposalSaleableItem.class)
  public class SaleableNegociated {
   		private Long id;
		private BigDecimal price;
		Product product;
		private Integer quantity;

		//getters and setters
   }

  @EntityReference(Product.class)
  public class Saleable {
    private Long id;
    private String name;
  }
  ```

  Tendo as entidades devidamente mapeadas agora eu posso receber os valores da interface, passar pelo meu serviço
  e antes de persistir eu preciso fazer a conversão para o objeto do banco de dados(hibernate)
  exemplo:

  ```javascript
  public class NegotiationService {

      private NegotiationRepository repository;

      public void save(Negotiation negotiation) {

          repository.save(negotiation)
      }

  }


  public interface NegotiationRepository {

      public void save(Negotiation negotiation);
  }

  public class NegotiationRepositoryHibernate implements NegotiationRepository {

      private BusinessProposalRepository repository;

      public void save(Negotiation negotiation) {
          BusinessProposal businessProposal = BusinessModelClone.from(negotiation).convertTo(BusinessProposal.class);
          repository.save(businessProposal)
      }

  }
  ```

  Agora suponha que já existe uma entidade BusinessProposal no bando de dados com o ID 1 então precisamos apenas
  atualizar-la para isso a implementacao do nosso repository seria assim:

  ```javascript
  //Implementacao para suporta updates
  public class NegotiationRepositoryHibernate implements NegotiationRepository {

      private BusinessProposalRepository repository;

      public void save(Negotiation negotiation) {
          if (negotiation.isNew()) {
            BusinessProposal businessProposal = BusinessModelClone.from(negotiation).convertTo(BusinessProposal.class);
            repository.save(businessProposal)
          } else {
              BusinessProposal businessProposalLoaded = repository.findOne(negotiation.getId());
              BusinessModelClone.from(negotiation).merge(businessProposalLoaded);

              repository.save(businessProposalLoaded);
          }

      }

  }
  ```

  Proxy: Um grande problema que faz com que os usuários acabem por usar entidades do hibernade dentro de seus dominos
  e o tornando um pojo é a dificuldade em criar uma camada anti corrupção. Esse camada deve ser criada idependente
  se o sistema é novo ou legado, você não pode ficar refem de alterações no banco de dados.

  O modulo PROXY dessa api permite que você busque objetos de negocio e em Runtime esses objetos de negocio peguem
  as informações da entidade do Hibernate usando o Lazzy Load

  Exemplo:

  ```javascript
    //Implementacao para o findOne
  public class NegotiationRepositoryHibernate implements NegotiationRepository {

      private BusinessProposalRepository repository;

      public Negotiation findOne(Long id) {
          BusinessProposal businessProposalLoaded = repository.findOne(id);

          Negotiation negotiationProxy = BusinessModelProxy.from(businessProposalLoaded).proxy(Negotiation.class);

          return negotiationProxy;
      }

  }
  ```

  Esse trecho de código irá buscar as informações no objeto do banco de dados(BusinessProposal) usando o lazzy load
  do hibernate;

  Quando eu fizer um negotitationProxy.getId() o proxy irá fazer um businessProposal.getId() e retornará o resultado.
  Se você fizer um negotitationProxy.getSeller() o proxy internamente irá ler as anotacoes, converter o resultado para um
  objeto de dominio e coloca-lo num proxy, como o exemplo abaixo:

    > //Internamente o proxy faz algo parecido com isso
    >  User user = businessProposal.getUser();
    >  Seller sellerProxy =  BusinessModelProxy.from(user).proxy(Seller.class);
    > return sellerProxy;

  No seu código irá aparece somente isso Seller seller = negotiationProxy.getSeller();


  Listas: O sistema de proxy suporta listas também, então ao fazer um negotiationProxy.getItemsNegotiated()
  o proxy irá pegar a lista de List<ProposalSaleableItem> items; do objeto do hibernate e converter para
  uma lista de List<SaleableNegociated> itemsNegotiated e assim você terá a sua lista do seu objeto de dominio
  de forma magica.

  Setter: Quando a entidade do hibernate é carregada usando um repository.findOne(1l) essa entidade está
  sendo gerenciada pelo hibernate e qualquer alteração que ela sofrer será refletida no banco de dados.
  O proxy do ddd-reflection também irá fornecer a mesma facilidade que o hibernate fornece.

  Quando você carrega o objeto do banco de dados e ele está dentro do proxy, exemplo:

  ```javascript
    //Implementacao para o findOne
  public class NegotiationRepositoryHibernate implements NegotiationRepository {

      private BusinessProposalRepository repository;

      public Negotiation findOne(Long id) {
          BusinessProposal businessProposalLoaded = repository.findOne(id);

          Negotiation negotiationProxy = BusinessModelProxy.from(businessProposalLoaded).proxy(Negotiation.class);

          return negotiationProxy;
      }

  }
  ```

  Quando eu carrego o objeto Negotitation negotiationLoaded = negotiationRepository.findOne(1l)
  e faço um negotiationLoaded.getSeller().setName("Jon Snow") o proxy irá fazer algo como

  businessProposalLoaded.getUser().setName("Jon Snow") então essa alteração irá refletir no banco
  de dados assim que seu framework fizer o commit antes de fechar a sessao do hibernate.

  O set do proxy também suporta objetos do seu dominio e listas, tudo isso é refletido na entidade do hibernate,
  mais um exemplo:

  > Negotitation negotiationLoaded = negotiationRepository.findOne(1l)