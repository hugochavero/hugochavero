# hugochavero
<html>
<header>
<title>Hugo Chavero</title>
</header>
<body>
<h1>Proyectos en los que he colaborado</h1>
<h3>API Rest para validación/generación de licencias para softwares de escritorio pertenecientes a Trascopier S.A.</h3>
<p>"
# -*- encoding: utf-8 -*-
from datetime import timedelta
import datetime
import json

from django.db.models import Q
from django.shortcuts import get_object_or_404
from rest_framework import viewsets, status
from rest_framework.decorators import detail_route
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

from servicemanager.backend.models import ContractOperation, Contract, Product, Client, \
    ProductRegistration
from servicemanager.backend.serializers import ContractSerializer, \
    ContractOperationSerializer, ProductRegistrationSerializer


# Validez de licencia DV gratis (en cantidad de dias)
DV_DAYS = 30
DV_VERSION_FREE = 1
DV_VERSION_LITE = 2
DV_VERSION_PRO = 3
# Corresponde al Seller datovariable_user
DV_SM_USER = 3


class ContractOperationViewSet(viewsets.ModelViewSet):
    serializer_class = ContractOperationSerializer
    queryset = ContractOperation.objects.none()

    def get_queryset(self):
        server = self.request.user.get_server
        operations = ContractOperation.objects.filter(result=0)
        return operations.filter(contract_server__server = server)
        

    @detail_route(methods=['post'])
    def update_result(self, request, pk=None):
        try:
            contract_operation = ContractOperation.objects.get(pk=pk)
            contract_operation.result = int(request.POST['result'])
            contract_operation.save()
            return Response(status=status.HTTP_200_OK)
        except Exception, e:
            return Response(status=status.HTTP_400_BAD_REQUEST) 
        

class ContractViewSet(viewsets.ModelViewSet):
    serializer_class = ContractSerializer
    queryset = Contract.objects.none()
    permission_classes = [IsAuthenticated]

    @detail_route(methods=['post'])
    def activate_license(self, request, pk=None):
        '''
        Funcion encargada de activar una licencia previamente generada
        '''
        r = {}
        mac_address_post = request.DATA['mac_address']
        code_product = request.DATA['code_product']
        client_id = request.DATA.get('client_id')
        product = get_object_or_404(Product, product_type__code=code_product)
        contract = get_object_or_404(Contract, uid=pk, product=product, client_id=client_id)
        info = json.loads(contract.contract_info)

        # Si licencia fue previamente activada, verifico su validez
        if 'mac_address' and 'activated' in info:
            try:
                activated = datetime.datetime.strptime(info['activated'],"%d/%m/%Y")
            except ValueError:
                activated = datetime.datetime.strptime(info['activated'],"%m/%d/%Y")
            # Compruebo que el mac_address coincida con el registrado
            if info['mac_address'] != mac_address_post:
                r['msg'] = u"Licencia registrada en otro Equipo.\
                                \nContacte a su Distribuidor."
                return Response(status=status.HTTP_403_FORBIDDEN, data=r)
            # Si tiene vencimiento por fecha
            if 'expire_date' in info:
                exp_date = datetime.datetime.strptime(info['expire_date'], '%d/%m/%Y') 
                if datetime.datetime.now() > exp_date :
                    r['msg'] = u"<p>Su licencia expiró el dia {}.\
                        <br><br>Solo podrá utilizar Gratis el Montador.\
                        <br><br>Para renovar su licencia diríjase a:\
                        <br><a href=http://www.datovariable.com.ar/contacto>\
                        http://www.datovariable.com.ar/contacto</a></p>".format(exp_date)
                    return Response(status=status.HTTP_403_FORBIDDEN, data=r)

            # Si tiene vencimiento por dias
            if 'expires_in' in info:
                expire = info['expires_in']
                now = datetime.datetime.today()
                delta = now - activated
                # Si la licencia expiró
                if delta.days > expire and bool(expire):
                    r['msg'] = u"<p>Licencia Expirada.\
                        <br><br>Solo podrá utilizar Gratis el Montador.\
                        <br><br>Para renovar su licencia diríjase a:\
                        <br><a href=http://www.datovariable.com.ar/contacto>\
                        http://www.datovariable.com.ar/contacto</a></p>"
                    return Response(status=status.HTTP_403_FORBIDDEN, data=r)
        # Activo licencia por primera vez
        else:
            info['mac_address'] = mac_address_post
            info['activated'] = datetime.date.today().strftime("%d/%m/%Y")

        # actualizo los siguientes datos
        info['state'] = request.DATA.get('state')
        info['web'] = request.DATA.get('web')
        info['phone'] = request.DATA.get('phone')

        # Retorno el modo de activacion de la licencia, sino tiene tiene dicha key
        # retornamos modo VERSION LITE ya que todas las licencias actuales corresponen
        # a dicha version
        if 'mode' not in info:
            info['mode'] = DV_VERSION_LITE
        contract.contract_info = json.dumps(info)
        contract.save()

        r['mode'] = info['mode']
        r['uid'] = str(contract.uid)
        r['msg'] = u"Licencia Validada Correctamente"
        return Response(status=status.HTTP_200_OK, data=r)
    
    @detail_route(methods=['post'], permission_classes=[IsAuthenticated])
    def check_license(self, request, pk=None):
        '''
        Funcion encargada de verificar el estado de la licencia que se provee
        '''
        contract = get_object_or_404(Contract, uid=pk)
        info = json.loads(contract.contract_info)
        now = datetime.datetime.today()
        r = {}
        
        r['mode'] = info['mode'] if 'mode' in info else DV_VERSION_LITE
        
        # Modo de vencimiento nuevo (a partir de 1/04/2017 se utiliza este modo)
        if 'expire_date' in info:
            try:
                expire_date = datetime.datetime.strptime(info['expire_date'],"%d/%m/%Y")
            except ValueError:
                expire_date = datetime.datetime.strptime(info['expire_date'],"%m/%d/%Y")
            if now > expire_date:
                r['msg'] = "Su licencia expiró el dia {}".format(expire_date)
                info['mode'] = DV_VERSION_FREE
            else:
                r['msg'] = "Licencia OK"
        else:
            # Modo de vencimiento viejo
            try:
                activated = datetime.datetime.strptime(info['activated'],"%d/%m/%Y")
            except:
                activated = datetime.datetime.strptime(info['activated'],"%m/%d/%Y")
            expire = info['expires_in']
            delta = now - activated
            if delta.days > expire and bool(expire):
                # Si la licencia expiró (metodo viejo)
                r['msg'] = "La licencia se encuentra expirada"
                info['mode'] = DV_VERSION_FREE
            else:
                r['msg'] = "Licencia OK"
                # Reemplazo el vencimiento por dias por vencimiento por fecha
                expire_date = activated + timedelta(days=3600)
                info['expire_date'] = expire_date.strftime('%d/%m/%Y')
                info.pop('expires_in')
        
        contract.contract_info = json.dumps(info)
        contract.save()
        return Response(data=r)
    
    '''
    @detail_route(methods=['post'])
    def set_contract_info(self, request, client_id=None):
        client = get_object_or_404(Client, id=client_id)
        contract = Contract.objects.filter(client=client)
        info = json.loads(contract.contract_info)
        info['city'] = request.POST['city']
        info['name'] = request.POST['name']
        info['address'] = request.POST['address']
        info['phone'] = request.POST['phone']
        info['state'] = request.POST['state']
        info['email'] = request.POST['email']
        contract.contract_info = json.dumps(info)
        c_name = request.POST['name']
        c_country = request.POST['country'].upper()
        created_by_id = contract.client.created_by_id
        client = Client.objects.create(name=c_name,
                                       country=c_country,
                                       created_by_id=created_by_id)
        contract.client = client
        contract.save()
        return Response(status=status.HTTP_200_OK)

    @detail_route(methods=['post'])
    def get_contract_info(self, request, client_id):
        contract = get_object_or_404(Contract, client_id=client_id)
        info = json.loads(contract.contract_info)
        return Response(data=info)
    '''

    @detail_route(methods=['post'])
    def get_license(self, request, pk):
        '''
        Funcion encargada de generar una licencia por DV_DAYS dias
        '''
        r = {}
        client_id = pk
        mac_address = request.DATA.get('mac_address')
        product_id = request.DATA.get('product_id')
        state = request.DATA.get('state')
        phone = request.DATA.get('phone')
        web = request.DATA.get('web')
        client = Client.objects.get(id=client_id)
        product = Product.objects.get(product_id=int(product_id))
	blank_contract = None
        # Verifico que esta mac no se encuentre vinculada a contratos de otros clientes
        for notc in Contract.objects.filter(~Q(client=client), product=product):
            info = json.loads(notc.contract_info)
            if 'mac_address' in info:
                if mac_address in info['mac_address']:
                    print notc.id
                    r['msg'] = "Ya existe una licencia otorgada a esta Computadora."
                    return Response(status=status.HTTP_403_FORBIDDEN, data=r)
        # Recorro los contratos existentes del usuario
        for c in Contract.objects.filter(client=client, product=product):
            info = json.loads(c.contract_info)
            # Si coincide la mac actual con alguno de los contratos verifico que todavia es valido
            # Si el actual contrato no tiene mac asignada, entonces lo populo
            if 'mac_address' in info and mac_address in info['mac_address']:
                if 'expire_date' in info:
                    exp_date = datetime.datetime.strptime(info['expire_date'], '%d/%m/%Y') 
                    if datetime.datetime.now() > exp_date :
                        r['msg'] = u"<p>Período de prueba Full Agotado.\
                        <br><br>Solo podrá utilizar Gratis el Montador.\
                        <br><br>Para extender DatoVariable Full diríjase a:\
                        <br><a href=http://www.datovariable.com.ar/contacto>\
                        http://www.datovariable.com.ar/contacto</a></p>"
                        return Response(data=r)
                    else:
                        r['msg'] = "Version Full Activada Exitosamente"
                        r['uid'] = str(c.uid)
                        r['mode'] = DV_VERSION_LITE
                        return Response(data=r)
                else:
                    return self.activate_license(request, str(c.uid))
            else:
                blank_contract = c
        # Si el usuario notiene contratos o si no se vincula la actual mac en ninguno
        # de los contratos existentes, entonces genero un nuevo contrato.
        ci = {}
        activated_date = datetime.datetime.now()
        expire_date = activated_date + timedelta(days=DV_DAYS)
        contract = blank_contract or Contract.objects.create(client=client, product=product, created_by_id=DV_SM_USER)
        p_registration = ProductRegistration.objects.get(client=client, product=product)
        p_registration.sate = state
        p_registration.phone = phone
        p_registration.web = web
        p_registration.save()
        ci['mac_address'] = mac_address
        ci['state'] = state
        ci['phone'] = phone
        ci['web'] = web
        ci['activated'] = activated_date.strftime('%d/%m/%Y')
        ci['expire_date'] = expire_date.strftime('%d/%m/%Y')
        ci['mode'] = DV_VERSION_LITE
        contract.contract_info = json.dumps(ci)
        contract.save()
        r['msg'] = "A partir de ahora tiene Montador gratis de por vida y\
        \nDatoVariable Full por {} dias.".format(DV_DAYS)
        r['uid'] = str(contract.uid)
        r['mode'] = DV_VERSION_LITE
        return Response(data=r)


class RegistrationViewSet(viewsets.ModelViewSet):
    queryset = ProductRegistration.objects.none()
    serializer_class = ProductRegistrationSerializer
    permission_classes = [IsAuthenticated]

    def create(self, request):
        '''
        Funcion encargada de crear un cliente y un registro de usuario.
        Estos clientes son creados por admin
        '''
        r = {}
        mac_address = request.DATA.get('mac_address')
        name = request.DATA.get('name')
        email = request.DATA.get('email')
        country = request.DATA.get('country')
        product_id = request.DATA.get('product')
        client = Client.objects.filter(email=email)
        if client.exists():
            client = client.get()
        else:
            client = Client.objects.create(name=name,
                                  created_by_id=DV_SM_USER
                                  )
            client.country.code = country
        product = Product.objects.get(product_id=int(product_id))
        reg_product, pr_created = ProductRegistration.objects.get_or_create(mac_address=mac_address)
        reg_product.client = client
        reg_product.email = request.DATA.get('email')
        reg_product.state = request.DATA.get('state')
        reg_product.phone = request.DATA.get('phone')
        reg_product.web = request.DATA.get('web')
        reg_product.mac_address = request.DATA.get('mac_address')
        reg_product.product_id = product
        reg_product.save()
        r['msg'] = "Muchas Gracias! Ya puede comenzar a utilzar {}".format(product.name)
        r['client_id'] = client.id
        return Response(data=r)"
</p>


</body>
</html>

<h1>header 1</h1>
<h2>header 2</h2>
<h3>header 3</h3>
<h4>header 4</h4>

