---
published: true
---


In  Symfony, When creating Form with single/multiple select from dropdown, we ussually have problem with a large amount of data. With [Select2](https://select2.org/), the jQuery replacement for select boxes, we can do it easily by using ajax. However, in certain specific cases, we have to do it for ourself instead of using pre-existing library/package. In this post, I'm going to share to you how to customize EntityType of Symfony2 with Select2. 

1. [Select2 3.5.4 ](https://github.com/select2/select2/tree/3.5.4) 
2. [Symfony 2.8](https://symfony.com/)

### Problem

Due to many reasons, you can not refactor/upgrade your project/framework/library... In my case, I have to resolve the problem with a large amount of data without upgrade Select2 from 3.5.4 to 4.x, there is an awesome php package called [select2entity-bundle](https://github.com/tetranz/select2entity-bundle) which supports symfony2 and symfony3. There are 2 problems:
1. The awesome options on v2.* only works with Select2 version 4.*, whereas [select2entity-bundle](https://github.com/tetranz/select2entity-bundle) v1.x works on select2 v3.x has limit options. 
2. I'd like to embed some extra parameters and csrf token to the ajax request to make the system more flexible and secure.

### How to do that

**Firstly**, create a FormType called `Select2EntityType` which extends `Symfony\Component\Form\AbstractType` like this:
    
    <?php
        use Doctrine\ORM\EntityManager;
        use Symfony\Component\Form\AbstractType;
        use Symfony\Component\Form\FormBuilderInterface;
        use Symfony\Component\Form\FormInterface;
        use Symfony\Component\Form\FormView;
        use Symfony\Component\OptionsResolver\OptionsResolver;
        use Symfony\Component\Routing\Router;
        use Symfony\Component\PropertyAccess\PropertyAccess;
        use DataTransformer\EntitiesToPropertyTransformer;
        use DataTransformer\EntityToPropertyTransformer;
    
        class Select2EntityType extends AbstractType
        {
            protected $em;
            protected $router;

            /**
            * Select2EntityType constructor.
            * @param EntityManager $em
            * @param Router        $router
            */
            public function __construct(EntityManager $em, Router $router) {
                $this->em = $em;
                $this->router = $router;
            }

            /**
            * @param FormBuilderInterface $builder
            * @param array                $options
            */
            public function buildForm(FormBuilderInterface $builder, array $options) {
                // add the appropriate data transformer
                $transformer = $options['multiple']
                    ? new EntitiesToPropertyTransformer($this->em, $options['class'], $options['text_property'])
                    : new EntityToPropertyTransformer($this->em, $options['class'], $options['text_property']);
                $builder->addViewTransformer($transformer, true);
            }

            /**
            * @param FormView      $view
            * @param FormInterface $form
            * @param array         $options
            */
            public function finishView(FormView $view, FormInterface $form, array $options) {
                parent::finishView($view, $form, $options);
                // make variables available to the view
                $view->vars['remote_path'] = $options['remote_path']
                    ?: $this->router->generate($options['remote_route'], $options['remote_params']);

                if (isset($options['req_params']) &&
                    is_array($options['req_params']) &&
                    count($options['req_params']) > 0) {
                    $accessor = PropertyAccess::createPropertyAccessor();
                    $reqParams = [];
                    foreach ($options['req_params'] as $key => $reqParam) {
                        $reqParams[$key] = $accessor->getValue($view, $reqParam . '.vars[full_name]');
                    }
                    $view->vars['attr']['data-req_params'] = json_encode($reqParams);
                }

                $varNames = [
                    'remote_method',
                    'multiple',
                    'minimum_input_length',
                    'page_limit',
                    'data_type',
                    'placeholder',
                    'csrf_token_class',
                    'request_optional_params',
                    'scroll',
                    'autostart',
                ];
                foreach ($varNames as $varName) {
                    $view->vars[$varName] = $options[$varName];
                }
            }

            /**
            * @param OptionsResolver $resolver
            */
            public function configureOptions(OptionsResolver $resolver) {
                $resolver->setDefaults([
                    'class' => null,
                    'remote_path' => null,
                    'remote_route' => null,
                    'remote_method' => 'POST',
                    'remote_params' => [],
                    'req_params' => [],
                    'multiple' => false,
                    'compound' => false,
                    'minimum_input_length' => 1,
                    'page_limit' => 100,
                    'data_type' => 'json',
                    'scroll' => true,
                    'autostart' => true,
                    'text_property' => null,
                    'placeholder' => '',
                    'csrf_token_class' => 'csrf_token_class',
                    'request_optional_params' => [],
                ]);
            }

            /**
            * @return string
            */
            public function getName() {
                return 'hd_select2entity';
            }
        }

**Secondly**, create DataTransformer classes:

**EntityToPropertyTransformer**

    use Doctrine\ORM\EntityManager;
    use Symfony\Component\Form\DataTransformerInterface;
    use Symfony\Component\PropertyAccess\PropertyAccess;

    class EntityToPropertyTransformer implements DataTransformerInterface
    {
        protected $em;
        protected $className;
        protected $textProperty;

        /**
        * EntityToPropertyTransformer constructor.
        * @param EntityManager $em
        * @param string        $class
        * @param string        $textProperty
        */
        public function __construct(EntityManager $em, $class, $textProperty) {
            $this->em = $em;
            $this->className = $class;
            $this->textProperty = $textProperty;
        }
        /**
        * Transform entity to json with id and text
        *
        * @param mixed $entity
        *
        * @return string
        */
        public function transform($entity) {
            if (null === $entity || '' === $entity) {
                return '';
            }
            $accessor = PropertyAccess::createPropertyAccessor();
            // return the initial values as html encoded json
            $text = is_null($this->textProperty)
                ? (string)$entity
                : $accessor->getValue($entity, $this->textProperty);
            $data = [
                'id' => $accessor->getValue($entity, 'id'),
                'text' => $text,
            ];

            return htmlspecialchars(json_encode($data));
        }
        /**
        * @param mixed $value
        *
        * @return mixed|null|object
        */
        public function reverseTransform($value) {
            if (null === $value || '' === $value) {
                return null;
            }
            $repo = $this->em->getRepository($this->className);

            return $repo->find($value);
        }
    }

**EntitiesToPropertyTransformer**

    use Doctrine\Common\Collections\ArrayCollection;
    use Doctrine\ORM\EntityManager;
    use Symfony\Component\Form\DataTransformerInterface;
    use Symfony\Component\PropertyAccess\PropertyAccess;

    class EntitiesToPropertyTransformer implements DataTransformerInterface
    {
        protected $em;
        protected $className;
        protected $textProperty;

        /**
        * EntitiesToPropertyTransformer constructor.
        * @param EntityManager $em
        * @param string        $class
        * @param string        $textProperty
        */
        public function __construct(EntityManager $em, $class, $textProperty) {
            $this->em = $em;
            $this->className = $class;
            $this->textProperty = $textProperty;
        }

        /**
        * @param mixed $entities
        *
        * @return mixed|string
        */
        public function transform($entities) {
            if (count($entities) == 0) {
                return '';
            }
            // return an array of initial values as html encoded json
            $data = array();
            $accessor = PropertyAccess::createPropertyAccessor();
            foreach ($entities as $entity) {
                $text = is_null($this->textProperty)
                    ? (string) $entity
                    : $accessor->getValue($entity, $this->textProperty);
                $data[] = [
                    'id' => $accessor->getValue($entity, 'id'),
                    'text' => $text,
                ];
            }

            return htmlspecialchars(json_encode($data));
        }

        /**
        * @param mixed $values
        *
        * @return array|ArrayCollection|mixed
        */
        public function reverseTransform($values) {
            // remove the 'magic' non-blank value added in fields.html.twig
            $values = ltrim($values, '-1,');
            if (null === $values || '' === $values) {
                return new ArrayCollection();
            }
            $ids = explode(',', $values);
            // get multiple entities with one query
            $entities = $this->em->createQueryBuilder()
                ->select('entity')
                ->from($this->className, 'entity')
                ->where('entity.id IN (:ids)')
                ->setParameter('ids', $ids)
                ->getQuery()
                ->getResult();
            return $entities;
        }
    }

**Fourth**, in app/src/AppBundle/Resources/views/form/fields.twig

    {% block hd_select2entity_widget %}
        {% set attr = attr|merge({
            'data-multiple': multiple ? 1 : 0,
            'data-min-length': minimum_input_length,
            'data-csrf-token-class': csrf_token_class,
            'data-request-optional-params': request_optional_params | json_encode(),
            'data-ajax--url': remote_path,
            'data-ajax--method': remote_method | default('POST'),
            'data-ajax--cache': 'true',
            'data-ajax--cache-timeout': 1000,
            'data-ajax--delay': 250,
            'data-ajax--data-type': "json",
            'data-language': 'en',
            'data-minimum-input-length': 1,
            'data-placeholder': placeholder|trans({}, translation_domain),
            'data-page-limit': page_limit,
            'data-scroll': scroll ? 'true' : '',
            'data-autostart': autostart ? 'true' : '',
        }) %}

        {# Add class for jQuery for find it #}
        {% set attr = attr|merge({ 'class': (attr.class|default('') ~ ' hdselect2entity')|trim }) %}

        {% set type = type|default('hidden') %}
        {{ block('form_widget_simple') }}
    {% endblock %}
    

**Firth**, in app/config/config.yml

    # Twig Configuration
    twig:
        form:
            resources:
                - 'AppBundle:form:fields.html.twig'

**Sixth**, in src/AppBundle/Resources/public/js/select2entity/select2entity.js

    (function( $ ) {
        $.fn.select2entity = function (options) {
            this.each(function () {
                var request;
                // Keep a reference to the element so we can keep the cache local to this instance and so we can
                // fetch config settings since select2 doesn't expose its options to the transport method.
                var $s2 = $(this),
                    scroll =  $s2.data('scroll');
                    limit = $s2.data('page-limit') || 0;

                var reqParams = $s2.data('req_params');
                if (reqParams) {
                    $.each(reqParams, function (key, value) {
                        $('*[name="'+value+'"]').on('change', function () {
                            $s2.val(null);
                            $s2.trigger('change');
                        });
                    });
                }

                // Deep-merge the options
                $s2.select2($.extend(true, {
                    ajax: {
                        url: $s2.data('ajax--url'),
                        dataType: $s2.data('ajax--data-type'),
                        type: $s2.data('ajax--method'),
                        quietMillis: $s2.data('ajax--delay'),
                        cache: $s2.data('ajax--cache'),
                        transport: function (params) {
                            var options = {
                                url: $s2.data('ajax--url'),
                                type: $s2.data('ajax--method')
                            };
                            params = $.extend(params, options);
                            // console.log('params --- ' + JSON.stringify(params));
                            if (request) {
                                request.abort();
                            }
                            request = $.ajax(params).fail(params.error).done(params.success).always(function () {
                                request = undefined;
                            });
                        },
                        data: function (term, page) {
                            var csrf_token_class=  $s2.data('csrf-token-class');
                            var token = $("." + csrf_token_class).val();
                            var requestOptionalParams = $s2.data('request-optional-params');
                            var ret = {
                                '_csrf_token' : token,
                                'q': term,
                                'page' : page,
                                'limit' : limit
                            };
                            if (requestOptionalParams) {
                                data = $.extend(ret, requestOptionalParams);
                            }
                            var reqParams = $s2.data('req_params');
                            if (reqParams) {
                                $.each(reqParams, function (key, value) {
                                    var fieldElement = $('*[name="'+value+'"]');
                                    var reqValue = $(fieldElement).val();
                                    if ($(fieldElement).data('multiple') === 'multiple' || $(fieldElement).data('multiple') == '1') {
                                        if (reqValue && Array.isArray(reqValue) === false) {
                                            reqValue = reqValue.replace(/\[(.+?)\]/g,""); // remove any content with []
                                            reqValue = reqValue.replace(/\s/g,''); // remove all whitespaces
                                            reqValue  = reqValue.split(',');
                                            reqValue = reqValue.filter(function (e) {
                                                return e;
                                            });
                                        }
                                    }
                                    ret[key] = reqValue;
                                });
                            }
                            // only send the 'page' parameter if scrolling is enabled
                            if (scroll) {
                                ret['page'] = page || 1;
                            }
                            return ret;
                        },
                        results: function (data, page, query) {
                            var results, more = false, response = {};
                            page = page || 1;
                            if ($.isArray(data)) {
                                results = data;
                            } else if (typeof data == 'object') {
                                // assume remote result was proper object
                                results = data.results;
                                more = data.more;
                            } else {
                                // fail safe
                                results = [];
                            }
                            if (scroll) {
                                response.more = more;
                            }
                            response.results = results;
                            return response;
                        }
                    }
                }, options || {}));
            });
            return this;
        };
    })( jQuery );

    (function( $ ) {
        $(document).ready(function () {
            $('.hdselect2entity[data-autostart="true"]').each(function(index) {
                var initValue;
                var multiple = $(this).data('multiple');
                var options = {
                    placeholder: $(this).data('placeholder'),
                    allowClear: true,
                    multiple: multiple,
                    closeOnSelect: false,
                    minimumInputLength: $(this).data('min-length'),
                    initSelection : function (element, callback) {
                        var value = $(element).val();
                        if (value === '') {
                            if (multiple) {
                                initValue = [];
                            }
                            else {
                                initValue = {id:'', text:''};
                            }
                        }
                        else {
                            value = htmlDecode(value);
                            initValue = JSON.parse(value);
                        }
                        callback(initValue);
                    }
                };
                $(this).select2entity(options);
                if (!multiple) {
                    // Sets the data as dirty. Without this, the data comes through as blank if a form is submitted but the field is not changed
                    $(this).select2('data', initValue);
                }

            });
            // simple html decode. Basic idea from underscore.js
            function htmlDecode(string)
            {
                var regex = new RegExp('(&amp;|&lt;|&gt;|&quot;|&#x27;)', 'g');
                var map = {
                    '&amp;': '&',
                    '&lt;': '<',
                    '&gt;': '>',
                    '&quot;': '"',
                    '&#x27;': "'"
                };
                return ('' + string).replace(regex, function(match) {
                    return map[match];
                });
            }

            /**
            *  Fix closeOnSelect not working when minimum input length >= 1
            **/
            $('.hdselect2entity[data-multiple="1"]').on("select2-selecting", function(e) {
                e.preventDefault();
                var target = $(this);
                handleSelection(e, target);
            });

            function handleSelection(e, target){
                e.preventDefault();
                var data = target.select2('data');
                data.push({
                    'id': e.object.id,
                    'text': e.object.text
                });
                target.select2('data', data);
                target.select2("open");
            }
        });
    })( jQuery );

### How to use it

**Requirement:** 
- Platform already defined a route for all ajax functions called `ajaxGetData`, any requests to that route will be handled in the function `dataSearchAction` in `AjaxController.php`, in this function, Platform validate `CsrfTokenValid` and call a function based on request parameter `function`. Then return the data which return by `requested function`. 

- Nike have some projects such as: Footwear, Manswear, Sportwear, Apparell... For every project, they have many suppliers in many regions. The relationship between project and supplier is presented in the Purchase Orders (PO). For example, Nike have 5 POs on project Footwear and 10 POs on project Manswear with  with Supplier 1. They have 10 POs on project Footwear and 10 POs on project Sportwear with  with Supplier 2... They would like to have a feature to create the notes. Every note should be belongs to 1 or many projects, 1 or many suppliers, none or many POs. So that, POs should be filtered by selected projects and suppliers.

    public function dataSearchAction(Request $request) {

        if (false === $this->hasToken($request)) {
            return new JsonResponse(
                [
                    "result" => false,
                    "message" => "Access Denied",
                    "token_name" => $this->token_name,
                    "token" => $request->request->get('_csrf_token')
                ]
            );
        }
        $function = $request->get('function');
        if (method_exists($this, $function)) {
            try {
                $ReflectionMethod =  new \ReflectionMethod(get_class($this),$function;
                $args = array();
                    $params = $ReflectionMethod->getParameters();
                    $paramNames = array_map(function ($item) {
                        return $item->getName();
                    }, $params);

                foreach ($paramNames as $name) {
                    $args[] = $request->get($name);
                }
                return $ReflectionMethod->invokeArgs($this, $args);
            } catch (Exception $e) {
                $logger = $this->get('logger');
                $logger->critical(
                    "Something went wrong", [
                            "message" => $e->getMessage()
                        ]
                    );
                $response = new AjaxResponse(false, "Something went wrong");
                return $response->toJson();
            }
        } else {
            $response = new AjaxResponse(false, "Function - $function - not found");
            return $response->toJson();
        }
    }


In FormType (eg : NoteType), buildForm:

    $builder->add('notePos', 'hd_select2entity', [
                'class' => Purchaseorders::class,
                'label' => "Purchase order(s)",
                'placeholder' => '(Select PO)',
                'text_property' => 'clientPoNo',
                'required' => false,
                'multiple' => true,
                'minimum_input_length' => 5,
                'remote_route' => 'ajaxGetData',
                'request_optional_params' => [
                        'function' => 'getPOsByProjectsAndSuppliers'
                    ],
                'req_params' => [
                    'projects' => 'parent.children[noteProjects]',
                    'suppliers' => 'parent.children[noteSuppliers]',
                ],
            ])

In noteType.twig, include select2entity.js
    
    {% block javascripts %}
        {{ parent() }}
        {% javascripts
        '@AppBundle/Resources/public/js/select2entity/select2entity.js'
        %}
        <script type="application/javascript" src="{{ asset_url }}"></script>
        {% endjavascripts %}
    {% endblock %}