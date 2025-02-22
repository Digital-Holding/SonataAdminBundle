Sortable Sonata Type Model in Admin
===================================

This is a full working example on how to implement a sortable
feature in your Sonata admin form between two entities.

Background
----------

The sortable function is already available inside Sonata for the ``ChoiceType``.
But the ``ModelType`` (or sonata_type_model) extends from choice, so this
function is already available in our form type. We only need some configuration
to make it work.

The goal here is to fully configure a working example to handle the following need :
User got some expectations, but some are more relevant than the others.

Pre-requisites
--------------

- you already have SonataAdmin and DoctrineORM up and running.
- you already have a ``UserBundle``.
- you already have ``User`` and ``Expectation`` Entities classes.
- you already have an ``UserAdmin`` and ``ExpectationAdmin`` set up.

The Recipe
----------

Part 1 : Update the data model configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The first thing to do is to update the Doctrine ORM configuration and
create the join entity between ``User`` and ``Expectation``. We are
going to call this join entity ``UserHasExpectations``.

.. note::

   We can't use a ``Many-To-Many`` relation here because the joined entity will required an extra field to handle ordering.

So we start by updating ``UserBundle/Resources/config/doctrine/User.orm.xml`` and adding a ``One-To-Many`` relation.

.. code-block:: xml

    <one-to-many field="userHasExpectations" target-entity="UserBundle\Entity\UserHasExpectations" mapped-by="user" orphan-removal="true">
        <cascade>
            <cascade-persist/>
        </cascade>
        <order-by>
            <order-by-field name="position" direction="ASC"/>
        </order-by>
    </one-to-many>

Then update ``UserBundle/Resources/config/doctrine/Expectation.orm.xml`` and also adding a ``One-To-Many`` relation.

.. code-block:: xml

    <one-to-many field="userHasExpectations" target-entity="UserBundle\Entity\UserHasExpectations" mapped-by="expectation" orphan-removal="false">
        <cascade>
            <cascade-persist/>
        </cascade>
    </one-to-many>

We now need to create the join entity configuration, create the following file in ``UserBundle/Resources/config/doctrine/UserHasExpectations.orm.xml``.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
                      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                      xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">

        <entity name="UserBundle\Entity\UserHasExpectations" table="user__expectations">
            <id name="id" type="integer">
                <generator strategy="AUTO"/>
            </id>
            <field name="position" column="position" type="integer">
                <options>
                    <option name="default">0</option>
                </options>
            </field>

            <many-to-one field="user" target-entity="UserBundle\Entity\User" inversed-by="userHasExpectations" orphan-removal="false">
                <join-column name="user_id" referenced-column-name="id" on-delete="CASCADE"/>
            </many-to-one>

            <many-to-one field="expectation" target-entity="UserBundle\Entity\Expectation" inversed-by="userHasExpectations" orphan-removal="false">
                <join-column name="expectation_id" referenced-column-name="id" on-delete="CASCADE"/>
            </many-to-one>
        </entity>
    </doctrine-mapping>

Part 2 : Update the data model entities
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Update the ``UserBundle\Entity\User.php`` entity with the following::

    /**
     * @var Collection|UserHasExpectations[]
     */
    protected $userHasExpectations;

    public function __construct()
    {
        $this->userHasExpectations = new ArrayCollection;
    }

    public function setUserHasExpectations(Collection $userHasExpectations): void
    {
        $this->userHasExpectations = new ArrayCollection;

        foreach ($userHasExpectations as $one) {
            $this->addUserHasExpectations($one);
        }
    }

    public function getUserHasExpectations(): Collection
    {
        return $this->userHasExpectations;
    }

    public function addUserHasExpectations(UserHasExpectations $userHasExpectations): void
    {
        $userHasExpectations->setUser($this);

        $this->userHasExpectations[] = $userHasExpectations;
    }

    public function removeUserHasExpectations(UserHasExpectations $userHasExpectations): void
    {
        $this->userHasExpectations->removeElement($userHasExpectations);
    }

Update the ``UserBundle\Entity\Expectation.php`` entity with the following::

    /**
     * @var Collection|UserHasExpectations[]
     */
    protected $userHasExpectations;

    /**
     * @param Collection|UserHasExpectations[] $userHasExpectations
     */
    public function setUserHasExpectations(Collection $userHasExpectations)
    {
        $this->userHasExpectations = $userHasExpectations;
    }

    /**
     * @return Collection|UserHasExpectations[]
     */
    public function getUserHasExpectations(): Collection
    {
        return $this->userHasExpectations;
    }

    /**
     * @return string
     */
    public function __toString()
    {
        return $this->getLabel();
    }

Create the ``UserBundle\Entity\UserHasExpectations.php`` entity with the following::

    namespace UserBundle\Entity;

    class UserHasExpectations
    {
        /**
         * @var int $id
         */
        protected $id;

        /**
         * @var User $user
         */
        protected $user;

        /**
         * @var Expectation $expectation
         */
        protected $expectation;

        /**
         * @var int $position
         */
        protected $position;

        public function getId(): ?int
        {
            return $this->id;
        }

        public function getUser(): ?User
        {
            return $this->user;
        }

        public function setUser(User $user): void
        {
            $this->user = $user;
        }

        public function getExpectation(): ?Expectation
        {
            return $this->expectation;
        }

        public function setExpectation(Expectation $expectation): void
        {
            $this->expectation = $expectation;
        }

        public function getPosition(): ?int
        {
            return $this->position;
        }

        public function setPosition(int $position): void
        {
            $this->position = $position;
        }

        /**
         * @return string
         */
        public function __toString()
        {
            return (string) $this->getExpectation();
        }
    }

Part 3 : Update admin classes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is a very important part, the admin class **should** be created for the join entity. If you don't do that, the field will never display properly.
So we are going to start by creating this ``UserBundle\Admin\UserHasExpectationsAdmin.php``::

    namespace UserBundle\Admin;

    use Sonata\AdminBundle\Admin\AbstractAdmin;
    use Sonata\AdminBundle\Datagrid\ListMapper;
    use Sonata\AdminBundle\Form\FormMapper;

    final class UserHasExpectationsAdmin extends AbstractAdmin
    {
        protected function configureFormFields(FormMapper $form): void
        {
            $form
                ->add('expectation', 'sonata_type_model', ['required' => false])
                ->add('position', 'hidden')
            ;
        }

        protected function configureListFields(ListMapper $list): void
        {
            $list
                ->add('expectation')
                ->add('user')
                ->add('position')
            ;
        }
    }

... and define the service in ``UserBundle\Resources\config\admin.xml``.

.. code-block:: xml

    <service id="user.admin.user_has_expectations" class="UserBundle\Admin\UserHasExpectationsAdmin">
        <tag name="sonata.admin" model_class="UserBundle\Entity\UserHasExpectations" manager_type="orm" group="UserHasExpectations" label="UserHasExpectations"/>
    </service>

Now update the ``UserBundle\Admin\UserAdmin.php`` by adding the ``sonata_type_model`` field::

    protected function configureFormFields(FormMapper $form): void
    {
        $form
            ->add('userHasExpectations', 'sonata_type_model', [
                'label'        => 'User\'s expectations',
                'query'        => $this->modelManager->createQuery('UserBundle\Entity\Expectation'),
                'required'     => false,
                'multiple'     => true,
                'by_reference' => false,
                'sortable'     => true,
            ])
        ;

        $form
            ->get('userHasExpectations')
            ->addModelTransformer(new ExpectationDataTransformer($this->getSubject(), $this->modelManager));
    }

There is two important things that we need to show here :
* We use the field ``userHasExpectations`` of the user, but we need a list of ``Expectation`` to be displayed, that's explain the use of ``query``.
* We want to persist ``UserHasExpectations``Entities, but we manage ``Expectation``, so we need to use a custom `ModelTransformer <https://symfony.com/doc/5.4/form/data_transformers.html>`_ to deal with it.

Part 4 : Data Transformer
^^^^^^^^^^^^^^^^^^^^^^^^^

The last (but not least) step is create the ``UserBundle\Form\DataTransformer\ExpectationDataTransformer.php``
to handle the conversion of ``Expectation`` to ``UserHasExpectations``::

    namespace UserBundle\Form\DataTransformer;

    final class ExpectationDataTransformer implements Symfony\Component\Form\DataTransformerInterface
    {
        /**
         * @var User $user
         */
        private $user;

        /**
         * @var ModelManager $modelManager
         */
        private $modelManager;

        public function __construct(User $user, ModelManager $modelManager)
        {
            $this->user         = $user;
            $this->modelManager = $modelManager;
        }

        public function transform($value)
        {
            if (!is_null($value)) {
                $results = [];

                /** @var UserHasExpectations $userHasExpectations */
                foreach ($value as $userHasExpectations) {
                    $results[] = $userHasExpectations->getExpectation();
                }

                return $results;
            }

            return $value;
        }

        public function reverseTransform($value)
        {
            $results  = new ArrayCollection();
            $position = 0;

            /** @var Expectation $expectation */
            foreach ($value as $expectation) {
                $userHasExpectations = $this->create();
                $userHasExpectations->setExpectation($expectation);
                $userHasExpectations->setPosition($position++);

                $results->add($userHasExpectations);
            }

            // Remove Old values
            $qb   = $this->modelManager->getEntityManager()->createQueryBuilder();
            $expr = $this->modelManager->getEntityManager()->getExpressionBuilder();

            $userHasExpectationsToRemove = $qb->select('entity')
                                               ->from($this->getClass(), 'entity')
                                               ->where($expr->eq('entity.user', $this->user->getId()))
                                               ->getQuery()
                                               ->getResult();

            foreach ($userHasExpectationsToRemove as $userHasExpectations) {
                $this->modelManager->delete($userHasExpectations, false);
            }

            $this->modelManager->getEntityManager()->flush();

            return $results;
        }
    }
